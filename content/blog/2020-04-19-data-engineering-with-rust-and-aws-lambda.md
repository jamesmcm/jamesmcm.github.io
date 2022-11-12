+++
title = "Data Ingestion with Rust and AWS Lambda"
date  = 2020-04-19
[taxonomies]
categories = ["Rust"]
+++

In this post we will set up a very simple data ingestion process with Rust and AWS Lambda.

The complete code for this example is [available on GitHub here](https://github.com/jamesmcm/rust-lambda-test).

<!-- more -->

Opinions expressed are solely my own and do not express the views or opinions of my employer.

You should be able to follow along using the AWS Free Tier if you want, but you will need to use a Postgres 11.1+ RDS instance instead of Redshift (unless you qualify for the separate 12-month free trial of Amazon Redshift). AWS Lambda has an Always Free tier option, whereas S3 and RDS have only 12-month free trial options.

## Process Requirements

In this process we would like to parse, filter and import an Excel file to our database, on-demand when it is uploaded to S3.

We can split the problem down in to the following steps:

* Parsing the Excel file and filtering rows - here we will use the [calamine](https://crates.io/crates/calamine) and [Serde](https://crates.io/crates/serde) crates, note we need to handle possible parsing errors.
* Serialising the data to a CSV - we will use the [csv crate](https://crates.io/crates/csv) with [Serde](https://crates.io/crates/serde).
* Uploading the CSV to S3 - we will use the [rusoto crate](https://crates.io/crates/rusoto) for interacting with AWS.
* Getting database credentials from AWS Secrets Manager - we will use the rusoto crate.
* Triggering the COPY from S3 in the Redshift/RDS instance - we will use the [postgres crate](https://crates.io/crates/postgres) and [OpenSSL](https://crates.io/crates/openssl).
* Running this process in AWS Lambda and handling the events received - we will use the [lambda\_runtime](https://crates.io/crates/lambda_runtime) and [aws\_lambda\_events](https://crates.io/crates/aws_lambda_events) crates.
* Cross-compiling and deploying the process - we need to build for the `x86_64-unknown-linux-musl` target so the binary will run on Amazon Linux.

## Why Rust?

One could argue that Python is more suitable for this task, since pandas can parse Excel files and write to a CSV directly (using boto3 to interact with AWS). However there are a few reasons why I think Rust is better suited to this sort of serverless computing:

* With Rust we can deploy a statically linked binary - i.e. we don't have any dependency on the runtime environment. In Python we would have to handle deploying the dependent pip packages with the Lambda artefact, for NumPy this is not trivial since it has compiled dependencies which would need to be cross-compiled or built on Amazon Linux (i.e. with CodeBuild) - Lambda Layers can automate this for popular dependencies, but it's still more work than a static binary (and results in a larger artefact to deploy too).

* With Rust we get full type checking and static typing, in this case our example is simple serialisation and de-serialisation. However, if we were to do much more processing of the de-serialised data, it would be very useful to have all of that being type checked to avoid possible runtime errors.

* Rust gives us greater control of performance, and the lambda\_runtime crate allows us to use Tokio for asynchronous operations. With serverless computing we are paying by execution time (and memory usage with respect to the size of the instance) so better performance can directly lead to cost savings.

* Rust offers a better, modern development experience. With rust-analyzer you can have immediate type-checking directly in your IDE/text editor. Unit testing is built-in and easy to use in the same source file. Automatic generation of documentation is built-in with rustdoc. You also don't need to deal with setting up virtual environments for dependencies.

## Implementation

We will follow the steps outlined above, starting with parsing the Excel file, which can be tested independently.

### Parsing the Excel file with Calamine

The Excel file we want to parse has the following structure in the `data` worksheet:

location | metric | value | date 
--- | --- | --- | --- 
UK | conversion\_rate | 0 | 2020-02-01
ES | conversion\_rate | 0.634 | 2020-02-01
DE | conversion\_rate | #N/A | 2020-02-01
FR | conversion\_rate | #N/A | 2020-02-01
UK | conversion\_rate | 0.723 | 2020-01-31


We only want to import rows which have the same date as the first row in the Excel file.

Note the possibility of #N/A and invalid values which we will need to handle.

This file will be uploaded to `s3://input-bucket-name/label/filename.xlsx` where the label allows multiple files to be uploaded (with different locations in each file).

See [the Github repo](https://github.com/jamesmcm/rust-lambda-test) for a test example.

#### Reading the Excel file

First let's write a test that will read an example of our Excel file to a buffer and create a calamine::Xlsx object from it. We do it like this (rather than using calamine's `open_workbook()` function to read directly from a file) so we can maintain the same interface when reading the Excel file from S3 directly.

```rust
use calamine::{Reader, Xlsx};

##[cfg(test)]
mod tests {
    use super::*;
    use std::fs::File;
    use std::io::Write;
    use std::path::PathBuf;

    #[test]
    fn test_local() {
        let mut buffer: Vec<u8> = Vec::new();
        let mut f = File::open(
            PathBuf::from(env!("CARGO_MANIFEST_DIR")).join(PathBuf::from(
                "tests/test_excel.xlsx",
            )),
        )
        .ok()
        .unwrap();
        f.read_to_end(&mut buffer).ok().unwrap();

        let excel = match Xlsx::new(Cursor::new(buffer)) {
            Ok(x) => x,
            Err(x) => panic!("{:?}", x),
        };

        // ...
    }
}
```

Here we use the `CARGO_MANIFEST_DIR` environment variable to get a relative path when we run `cargo test`.

If you create an example Excel file, this test should pass. This provides us with a calamine::Xlsx object we can use to call our parsing function.

#### Parsing the Excel file

We will implement the following function:

```rust
fn excel_to_csv_string(
    mut excel: Xlsx<Cursor<Vec<u8>>>,
) -> Result<(String, HashSet<String>, chrono::NaiveDate), Box<dyn std::error::Error>>
```

Which will return a (Result of a) tuple of the CSV as a String, a HashSet of all found locations (for use in deleting existing rows before the database insertion), and the date of the first row that we used as the date of the file (also for use in the database insertion and logging).

Note the Xlsx object we pass is based on a Cursor in a Vec of u8 bytes - exactly what we generated above (and what we will later receive when reading the file on S3 with rusoto).

To do this, we will use calamine::RangeDeserializerBuilder, in our case since the Excel file has headers, we will use the `with_headers()` method and pass it a slice of static strings.

Let's first define the columns and the struct that we will deserialise the rows into:

```rust
use serde::{Deserialize, Serialize};

const COLUMNS: [&str; 4] = ["location", "metric", "value", "date"];

##[derive(Serialize, Deserialize, Debug)]
struct RawExcelRow {
    location: String,
    metric: String,
    #[serde(deserialize_with = "de_opt_f64")]
    value: Option<f64>,
    #[serde(deserialize_with = "de_date")]
    date: chrono::NaiveDate,
}
```

Our columns array corresponds to the headers in the Excel file (you could probably read these dynamically, but in our case we have a fixed schema anyway).

The RawExcelRow is the struct we will deserialise each row to. Note the `deserialize_with` field attributes set for the `value` and `date` fields. We will need to implement both of these.

The custom deserialiser for `value` is required so that we when the field fails to parse we replace the field value with None (rather than skipping the entire row, as we would if we handled the Error later when filtering rows).

This is implemented as follows (note we also explicitly cast integers, for the case of 0 or 1):

```rust
fn de_opt_f64<'de, D>(deserializer: D) -> Result<Option<f64>, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let data_type = calamine::DataType::deserialize(deserializer);
    match data_type {
        Ok(calamine::DataType::Error(_)) => Ok(None),
        Ok(calamine::DataType::Float(f)) => Ok(Some(f)),
        Ok(calamine::DataType::Int(i)) => Ok(Some(i as f64)),
        _ => Ok(None),
    }
}
```

For `date` the custom deserialiser is required since Excel stores dates as the number of days since January 1st 1900 (starting from 1).

Calamine has a function to do this conversion for us, but it won't automatically apply it when trying to deserialise to a chrono::NaiveDate (I created [this issue](https://github.com/tafia/calamine/issues/168) regarding this). 

So we simply call Calamine's `as_date()` conversion function explicitly:

```rust
fn de_date<'de, D>(deserializer: D) -> Result<chrono::NaiveDate, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let data_type = calamine::DataType::deserialize(deserializer);
    match data_type {
        Ok(x) => x.as_date().ok_or(serde::de::Error::custom("Invalid Date")),
        Err(x) => Err(x),
    }
}
```

With this in place we can implement the deserialisation step:

```rust
// ...
use calamine::{RangeDeserializerBuilder, Reader, Xlsx};
use csv::Writer;
use std::collections::HashSet;
use std::error::Error;
use std::io::Cursor;
use std::io::Read;
// ...

fn excel_to_csv_string(
    mut excel: Xlsx<Cursor<Vec<u8>>>,
) -> Result<(String, HashSet<String>, chrono::NaiveDate), Box<dyn std::error::Error>> {
    let range = excel
        .worksheet_range("data")
        .ok_or(calamine::Error::Msg("Cannot find data worksheet"))??;

    let mut iter_result =
        RangeDeserializerBuilder::with_headers(&COLUMNS).from_range::<_, RawExcelRow>(&range)?;

  // ...
```

Note the double `??` on the line `.ok_or(calamine::Error::Msg("Cannot find data worksheet"))??;` - this is used since
the `.worksheet_range()` method returns an `Option<Result<Range<DataType>, XlsxError>>` ([see docs](https://docs.rs/calamine/0.16.1/calamine/struct.Xlsx.html#method.worksheet_range)), 
and we convert the Option to a Result using `.ok_or()` ([see docs](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or)) to convert the
None case to a `calamine::Error` (and the Some case to Ok). The double `??` unwraps both of these
errors to give us the `Range<DataType>`.

We start by getting the range for the `data` worksheet, and then we deserialise it to `RawExcelRow`s, giving us an Iterator over `RawExcelRow`s. With this iterator we can apply row-level filtering (or other transformations), as we serialise to CSV.

#### Serialisation to CSV

We serialise the first row, and copy the date from it to filter the further rows (remember we want to filter only for rows which have the same date as the first row in the file).

```rust
    // ...
    // Use date of first row as date for file
    let mut wtr = Writer::from_writer(vec![]);
    let mut locations: HashSet<String> = HashSet::new();

    let first_row = iter_result.next().unwrap()?;
    let canonical_date = first_row.date.clone();
    locations.insert(first_row.location.clone());
    wtr.serialize(first_row)?;
    println!("Canonical date: {:?}", canonical_date);
    // ...
```

We then serialise all rows where the date is equal to the date of the first row (`canonical_date`), and add their `location` to the `locations` HashSet.

Finally, we convert the Writer object to a String, and return that, along with the `locations` HashSet and the `canonical_date`.

```rust
    // ...
    for (index, row) in iter_result.enumerate() {
        match row {
            Ok(row) => {
                if row.date == canonical_date {
                    locations.insert(row.location.clone());
                    wtr.serialize(row)?;
                }
            }
            Err(row) => println!("{}: {:?}", index, row),
        }
    }

    let data = String::from_utf8(wtr.into_inner()?)?;

    Ok((data, locations, canonical_date))
}
```


We can now test calling this function in our `test_local()` test, adding the following lines:

```rust
        // ...

        let (data, locations, canonical_date) = match excel_to_csv_string(excel) {
            Ok(x) => x,
            Err(x) => panic!("{:?}", x),
        };
        println!("locations: {:?}", locations);
        println!("canonical date: {:?}", canonical_date);
        let mut file = File::create(
            PathBuf::from(env!("CARGO_MANIFEST_DIR")).join(PathBuf::from("tests/test_output.csv")),
        )
        .ok()
        .unwrap();
        file.write_all(data.as_bytes()).ok().unwrap();
        assert_eq!(true, true);
    }
```

This test should pass, and write the CSV to `test_output.csv`.


### Uploading the CSV to S3

We can upload the CSV to S3 using the rusoto\_s3 crate.

```rust
use rusoto_core::Region;
use rusoto_s3::{GetObjectRequest, PutObjectRequest, S3Client, S3};
// ...
const OUTPUT_BUCKET: &str = "output-bucket-name";
// ...

fn upload_csv_to_s3(
    data: String,
    label: &str,
    canonical_date: &chrono::NaiveDate,
) -> Result<(String), Box<dyn std::error::Error>> {
    let outputkey = format!(
        "output/{}_{}.csv",
        label,
        canonical_date.format("%Y-%m-%d").to_string()
    );

    // Write CSV to S3
    let s3_client = S3Client::new(Region::EuWest1);
    s3_client
        .put_object(PutObjectRequest {
            bucket: String::from(OUTPUT_BUCKET),
            key: outputkey.clone(),
            body: Some(data.into_bytes().into()),
            ..Default::default()
        })
        .sync()?;

    Ok(outputkey)
}
```

Remember to set the correct region for your use case. In this case we use the default credentials provider, so your AWS credentials will also need to be available (i.e. in `~/.aws/credentials`) or via the runtime environment.

Note that here we consume our CSV String since we don't need it again. We also return the outputkey so we can use it later when triggering the load to the database.

We will get the `label` value from the path of the input file on S3 when the Lambda function is triggered. This allows for different files to be uploaded for the same day (i.e. splitting files by location).

Also note the call to `.sync()` since the result is a RusotoFuture object.

### Getting database credentials from AWS Secrets Manager

In order to connect to our database and trigger the load from S3, we first need to get the credentials.

Using AWS Secrets Manager is a best practice, since it allows you to easily rotate credentials whilst storing and sharing them securely. 

We will use the rusoto\_secretsmanager crate to retrieve our credentials.

```rust
use rusoto_core::Region;
use rusoto_secretsmanager::{GetSecretValueRequest, SecretsManager, SecretsManagerClient};
use serde::Deserialize;
use serde::Serialize;
// ...
##[allow(non_snake_case)]
##[derive(Serialize, Deserialize, Debug)]
struct DBCredentials {
    username: String,
    password: String,
    engine: String,
    host: String,
    port: u32,
    dbClusterIdentifier: String,
}
// ...

fn get_db_credentials() -> Result<DBCredentials, Box<dyn std::error::Error>> {
    let sm_client = SecretsManagerClient::new(Region::EuWest1);
    let secret = sm_client
        .get_secret_value(GetSecretValueRequest {
            secret_id: "db_credentials_secret".to_string(),
            version_id: None,
            version_stage: None,
        })
        .sync()?;

    let credentials: DBCredentials = serde_json::from_str(&secret.secret_string.unwrap())?;

    Ok(credentials)
}
```

Note here we have hardcoded the secret name `db_credentials_secret` and the AWS region.

We use the `serde_json` crate to deserialise the JSON returned from Secrets Manager into a DBCredentials struct we can use directly.

### Load the CSV to the database

#### Connecting to the database

Here we assume connecting to a Redshift cluster with SSL enabled. The requirement for SSL complicates things, here we will use OpenSSL.

```rust
use openssl::ssl::{SslConnector, SslMethod};
use postgres::Client;
use postgres_openssl::MakeTlsConnector;
// ...

fn load_to_db(
    outputkey: &str,
    canonical_date: &chrono::NaiveDate,
    locations: &HashSet<String>,
) -> Result<(), Box<dyn std::error::Error>> {

    let mut builder = SslConnector::builder(SslMethod::tls())?;
    builder.set_ca_file("redshift-ssl-ca-cert.pem")?;
    let connector = MakeTlsConnector::new(builder.build());

    let credentials = get_db_credentials()?;
    let mut client = Client::connect(
        format!(
            "host={} port={} dbname={} user={} password={} sslmode=require",
            credentials.host,
            credentials.port,
            "dbname",
            credentials.username,
            credentials.password
        )
        .as_str(),
        connector,
    )?;

    // ...
```

Note we have hardcoded the database name `dbname`.

The Redshift SSL CA certificate is available from AWS at [https://s3.amazonaws.com/redshift-downloads/redshift-ssl-ca-cert.pem](https://s3.amazonaws.com/redshift-downloads/redshift-ssl-ca-cert.pem)

This file should be in the root directory of the crate to work with the above code, and we will also need to add it to the artefact we upload to AWS Lambda.

Note in order to get this to cross-compile, we need to enable the `vendored` feature in the openssl crate, i.e. in Cargo.toml:

```toml
[dependencies]
postgres = "0.17.2"
postgres-openssl = "0.3.0"
openssl = {version = "0.10.28", features = ["vendored"]}
```

#### Loading the data

Now we have a working connection with SSL, we can load the data from S3. Continuing the above code:

```rust
    // ...
    let locations_vec: Vec<String> = locations
        .iter()
        .cloned()
        .map(|x| format!("'{}'", x))
        .collect();

    let target_table = "test_table";

    let location_string = &locations_vec.join(",");
    let truncate_query = format!(
        "DELETE FROM public.{} WHERE date = '{}' AND location IN ({});",
        target_table,
        canonical_date.format("%Y-%m-%d").to_string(),
        location_string
    );
    let colstr = &COLUMNS.join(",");
    println!("{}", truncate_query);
    let copy_query = format!(
        "COPY public.{} ({}) from
                 's3://{}/{}'
                  iam_role 'arn:aws:iam::YOUR_ROLE_HERE'
                  FORMAT CSV
                  EMPTYASNULL
                  BLANKSASNULL
                  IGNOREHEADER 1
                  IGNOREBLANKLINES
                  ;",
        target_table, colstr, OUTPUT_BUCKET, outputkey
    );
    println!("{:?}", client.execute(truncate_query.as_str(), &[]));
    println!("{:?}", client.execute(copy_query.as_str(), &[]));

    Ok(())
}
```

We have hardcoded the table name as `test_table`. We first delete from the table all rows which have the same date for each of the locations that we will insert. Then we trigger the load from S3 with the `COPY` statement.

Note you will need to use an appropriate IAM role in that statement.

### Running on AWS Lambda

We are now able to run the entire process locally, with a local Excel file, however in order to deploy it to AWS Lambda we need to make a few changes.

#### Loading the input Excel from S3

First of all, we need to be able to load the input Excel from S3. Specifically: given an S3 key, return the calamine::Xlsx object.

```rust
use calamine::{RangeDeserializerBuilder, Reader, Xlsx};
use rusoto_core::Region;
use rusoto_s3::{GetObjectRequest, PutObjectRequest, S3Client, S3};
use std::io::Cursor;
use std::io::Read;
// ...

fn get_excel_from_s3(
    bucket: &str,
    key: &str,
) -> Result<Xlsx<Cursor<Vec<u8>>>, Box<dyn std::error::Error>> {
    let mut buffer: Vec<u8> = Vec::new();
    let s3_client = S3Client::new(Region::EuWest1);

    println!("Reading bucket: {}, key: {}", bucket, key);
    let s3file = s3_client
        .get_object(GetObjectRequest {
            bucket: bucket.to_string(),
            key: key.to_string(),
            ..Default::default()
        })
        .sync()?;

    let _file = s3file
        .body
        .unwrap()
        .into_blocking_read()
        .read_to_end(&mut buffer)?;
    Ok(Xlsx::new(Cursor::new(buffer))?)
}
```

This will return us the `Xlsx<Cursor<Vec<u8>>` object that we want.

#### Bringing it all together

We now need to write the function that will call all of the above functions to run the process.

```rust
use calamine::{RangeDeserializerBuilder, Reader, Xlsx};
// ...
const INPUT_BUCKET: &str = "input-bucket-name";
// ...

fn handle_excel(key: &str) -> Result<(), Box<dyn std::error::Error>> {
    let label = key.split("/").nth(1).unwrap();
    let excel: Xlsx<_> = get_excel_from_s3(INPUT_BUCKET, &key)?;

    let (data, locations, canonical_date) = excel_to_csv_string(excel)?;

    let outputkey = upload_csv_to_s3(data, label, &canonical_date)?;

    load_to_db(&outputkey, &canonical_date, &locations)?;
    Ok(())
}
```

Remember the input file will be of the form: `s3://input-bucket-name/label/filename.xlsx`

#### Adding the lambda handler

Finally we need to add the functions which will be the entry point for the Lambda function:

```rust
use lambda_runtime::error::HandlerError;
use percent_encoding::percent_decode_str;
// ...

fn main() -> Result<(), Box<dyn Error>> {
    lambda_runtime::lambda!(my_handler);

    Ok(())
}

fn my_handler(
    e: aws_lambda_events::event::s3::S3Event,
    _c: lambda_runtime::Context,
) -> Result<(), HandlerError> {
    println!("{:?}", e);
    let decodedkey = percent_decode_str(&(e.records[0].s3.object.key.as_ref()).unwrap())
        .decode_utf8()
        .unwrap();

    match handle_excel(&decodedkey) {
        Ok(_) => (),
        Err(error) => {
            panic!("Error: {:?}", error);
        }
    }

    Ok(())
}
```

The events the Lambda will receive will come from S3 directly (i.e. forwarding PutObject events to this Lambda from the S3 bucket properties).

We use `aws_lambda_events` to get a template for this event, from which we extract the S3 key of the file uploaded which has triggered the function.

Note the use of the `percent_encoding` crate to decode the URL-encoded key we receive in the event itself.

The code for this example is [available on GitHub here](https://github.com/jamesmcm/rust-lambda-test).

### Deployment

To deploy the function, we follow the instructions on [the AWS blog about the Rust runtime](https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/).

We first need to edit Cargo.toml to set the binary name to `bootstrap`:

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"
```

#### OS X cross-compilation

If you are building on OS X you need to cross-compile the binary, as per the instructions in the blog post.

```bash
rustup target add x86_64-unknown-linux-musl
brew install filosottile/musl-cross/musl-cross
mkdir .cargo
echo '[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"' > .cargo/config
ln -s /usr/local/bin/x86_64-linux-musl-gcc /usr/local/bin/musl-gcc
```

Note the installation of musl-cross took almost 2 hours on my machine.

#### Build zip artefact

Remember to add the Redshift CA certificate to the zip archive:

```bash
cargo build --release --target x86_64-unknown-linux-musl
zip -j rust.zip ./target/x86_64-unknown-linux-musl/release/bootstrap ./redshift-ssl-ca-cert.pem
```

#### Upload to AWS

Create a new Lambda function with a Custom runtime, and then upload the zip file.

Note if your Redshift cluster (or RDS instance) is behind a VPC you will need to add the Lambda function to the same VPC. See [the documentation](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html) for more details. 

Then add the S3 trigger to the Lambda function (and you can test the function using the S3 event template).

## Future Improvements


### RusTLS

The dependency on OpenSSL can make it difficult to cross-compile, so it would be nice to use a pure Rust TLS library.

A good candidate would be [RustTLS](https://crates.io/crates/rustls), and the [tokio-postgres-rustls](https://crates.io/crates/tokio-postgres-rustls) crate apparently adds support for this to the postgres crate. 

I tried [native-tls](https://crates.io/crates/native-tls) which has built-in support in the postgres crate, but hit [this issue when building on OS X](https://github.com/sfackler/rust-native-tls/issues/132), and this utlimately uses OpenSSL on Linux anyway.

### Apache Arrow + FlatBuffers 

It would be great if we could deserialise directly (via Calamine and Serde) from the rows of the Excel file to a columnar form in memory such as an Apache Arrow [RecordBatch](https://arrow.apache.org/docs/r/reference/RecordBatch.html) or [Table](https://arrow.apache.org/docs/cpp/tables.html) (rather than deserialising to our `RawExcelRow` structs). This would allow us to use [DataFusion](https://github.com/apache/arrow/tree/master/rust/datafusion) to query the loaded data.

Ideally we could also apply transformations to the data column-wise, and add or remove columns, transitioning between different defined schemas. So we could have a data manipulation tool as powerful as pandas, but with statically defined schemas where the interactions can be verified at compile time.

This could then be saved to disk in the Apache Arrow format without serialisation, or rapidly serialised to columnar formats such as Parquet (which is also well-suited for loading to a columnar database).

Unfortunately, even though [Arrow does have a Rust implementation](https://github.com/apache/arrow/tree/master/rust), it doesn't seem this direct deserialisation with Serde is possible at the moment (although I am not very familiar with Apache Arrow, nor with Serde's internals).

An alternative could be to deserialise to an [ndarray](https://docs.rs/ndarray/0.13.1/ndarray/), however [support for Arrow is still in progress](https://github.com/rust-ndarray/ndarray/issues/771).

[FlatBuffers](https://google.github.io/flatbuffers/) (used for Arrow's Table) [also has an issue open for Serde serialisation](https://github.com/google/flatbuffers/issues/5132).

If you have any ideas for how to achieve this direct deserialisation with Serde please add a comment here in Disqus or as a [Github issue](https://github.com/jamesmcm/rust-lambda-test).

### Compressed Parquet output

As mentioned above, it'd be great to be able to write a columnar format such as Parquet directly for loading to Redshift.

Unfortunately it seems serialising to Parquet is not currently supported in Serde, although the [Amadeus](https://github.com/constellation-rs/amadeus) crate apparently supports reading Parquet.


## Conclusion

Thanks for reading this blog post, I hope this toy example is useful, and that Rust adoption will continue to grow in serverless computing.

If you questions or comments, or notice any issues in the examples, please create an issue on [the Github repo](https://github.com/jamesmcm/rust-lambda-test) or leave a comment here in Disqus.
