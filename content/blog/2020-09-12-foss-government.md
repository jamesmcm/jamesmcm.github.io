+++
title = "Why governments should adopt and invest in FOSS"
date = 2020-09-12
[taxonomies]
categories = ["Linux"]
+++

This post covers why I believe local and national governments should
adopt and invest in [Free and Open Source Software](https://www.gnu.org/philosophy/free-sw.en.html) (FOSS).

This has been in the news recently due to the city of Munich renewing 
it's [LiMux Linux distribution](https://www.zdnet.com/article/linux-not-windows-why-munich-is-shifting-back-from-microsoft-to-open-source-again/)
thanks to an agreement between the local SDP and Green politicians, and
the efforts of the [Public Money, Public Code](https://publiccode.eu/) campaign.

<!-- more -->

Opinions expressed are solely my own and do not express the views or opinions of my employer.

## LiMux

[LiMux](https://en.wikipedia.org/wiki/LiMux) is a project by the German city of Munich to migrate its desktop
infrastructure to FOSS solutions, specifically to the LiMux Linux
distribution (based on Ubuntu) and LibreOffice (initially to OpenOffice). The project was started in 2004, and
migration began in 2006. 

By the end of 2013, 15,000 desktops had been migrated successfully.
However, in September 2016 [Microsoft announced the opening of a new headquarters in Munich](https://mspoweruser.com/microsoft-germany-moves-into-a-new-headquarters/), and then
in 2017 the [city council decided to switch back to Microsoft Windows and Office](https://www.techrepublic.com/article/end-of-an-open-source-era-linux-pioneer-munich-confirms-switch-to-windows-10/).
The timing of this decision led to many accusations of effective
bribery, using the potential office opening to pressure the council in to using
Microsoft products (note that there is no evidence of a kick-back scheme
[like practiced by Microsoft Hungary](https://www.zdnet.com/article/microsoft-to-pay-25m-to-doj-and-sec-in-hungary-bribery-case/)).

Fortunately [this decision has now been reversed](https://fsfe.org/news/2020/news-20200506-01.en.html), and LiMux 
development and roll-out will continue.

## Other cities

Munich is not the only city choosing Free Software solutions, many
others are beginning to see the potential benefits too. In Catalonia,
Barcelona's [Digital Transformation programme supports Free Software](https://ajuntament.barcelona.cat/digital/es/transformacion-digital/tecnologia-para-un-gobierno-mejor/software-libre).
From first-hand experience this means that the government transport /
road organisation use LibreOffice for all documents - so local driving
schools, etc. do not need a copy of Microsoft Office to handle the
necessary digital paperwork.

In Valencia, [the local government migrated its administration and
schools to LibreOffice](https://www.muylinux.com/2013/08/22/generalitat-valenciana-libreoffice/)
saving 1.5 million euros per year, and saving students and other
end-users from having to purchase their own copies of Microsoft Office.

However, it is not always straightforward. [Munich faced a lot of push-back in some areas](https://lwn.net/Articles/737818/), and the [city of Barcelona still pays almost 5 million
euros](https://cronicaglobal.elespanol.com/politica/colau-gasta-4-5-millones-en-microsoft-mientras-promete-software-libre_32165_102.html)
in license costs to Microsoft.

The main issue here is not so much the direct cost to the city itself
(which is substantial, but not critical compared to other costs -
especially since any migration will incur a significant short-term
cost), but more that the cost must be paid over and over again by all
users separately. Barcelona continues to pay millions to Microsoft every
year, but so will other cities in Spain and across Europe, and then so will all
users of those services that may have to use a copy of Microsoft Office for
compatibility reasons (i.e. students and small businesses).

The same cost is paid over and over again to a foreign company that will
not hire local developers or invest in the local economy, and [uses vendor lock-in](https://en.wikipedia.org/wiki/Criticism_of_Microsoft#Vendor_lock-in)
to eventually control all of the critical infrastructure.

## Benefits of FOSS

The most commonly cited benefit of Free Software is certainly the cost
savings vs. proprietary licensing. Whilst this can be a benefit
in some circumstances, I think it overlooks many greater far-reaching
benefits, and may not actually manifest itself in the short-term due to
the additional costs of the actual migration (i.e. costs of deployment
and training).

### Shared investment

The main benefit of using FOSS solutions is that any investment in the
development and improvement of the software also benefits all other
users of the software.

That is, whilst the city of Munich might need [some improvements in
handling templates](https://github.com/WollMux/WollMux), other cities
might need other improvements - however they can all benefit from
each others' investment, with no need to pay again just to keep a license.

This helps to build up a commonwealth of high-quality, well-maintained
software which anyone can use. There is no central, unique
owner of the projects (any project could be forked if necessary), so
there is no company that can demand payments just to allow you to keep using the
existing software.

Any investment in development made is truly an investment, leading directly
to project improvements, rather than just money sent to a foreign
company whilst the customers have no ownership of the actual product,
allowing the company to charge even more in the future just for the
right to keep using it (usually this is done by giving discounts for the
early migration which then result in higher prices later once the
customer is locked in).

Note that this initial investment might be higher than the current costs
of existing proprietary software licensing, when considering the
migration and training costs and possible development costs. However,
unlike those licensing costs it is a one-time cost, and the customer
retains some ownership over the resulting product (i.e. there is not a
license that can be revoked).


### Freedom for development

There is also more freedom in the development process. The fact that the
products aren't owned by a company means that contributions can be made
by anyone, anywhere. 

Users of the software (such as the cities mentioned above) could fund
[local co-operatives of developers](https://github.com/hng/tech-coops) to add desired features and maintain
the projects, instead of being forced to fund a foreign corporation with
no local investment.

In the long-term this might also help to break up some of the monopolistic
Big Tech companies, and result in a freer society and business
environment for everyone.

### National security

In Europe especially, the current dependence on proprietary software
often means a dependence on foreign corporations which operate in
co-operation with their country's intelligence service. As revealed by
Edward Snowden, [Microsoft provided backdoor access to encrypted messages](https://www.theguardian.com/world/2013/jul/11/microsoft-nsa-collaboration-user-data) to
the NSA, CIA and FBI as part of the [PRISM programme](https://en.wikipedia.org/wiki/PRISM_(surveillance_program)).

It is indefensible that such a company can run the critical infrastructure
of the vast majority of local and central government administrations, as
well as personal computers. Especially when the [NSA has been proven to
intercept communications of supposedly allied nations](https://www.theguardian.com/us-news/2015/jul/08/nsa-tapped-german-chancellery-decades-wikileaks-claims-merkel).

With FOSS software there is no central company to be pressured by
intelligence services or build in backdoors. All of the code can be
scrutinised by developers and users. [Linus's Law](https://en.wikipedia.org/wiki/Linus%27s_law) states:

> Given enough eyeballs, all bugs are shallow.

The same concept applies to introducing backdoors to access private data. It
might be possible to pressure or bribe one company to do so, but it
isn't possible to do the same to tens of thousands of independent users and
developers.


## Bigger picture

The effects of co-ordinated FOSS adoption are much more far-reaching and
positive than solely saving the costs of current software licenses.

The adoption of FOSS and investment by many governments (at all levels)
would help to create a commonwealth of quality Free Software for all
citizens and government bodies to benefit from.

It would also allow development of these projects to be done from
anywhere, allowing users to fund local developers or co-operatives. This
would help to break up the massive, monopolistic Big Tech companies, and
could also be used as a way of bringing investment and jobs to parts of
the country/region that require it.

This would also be highly beneficial to developers, as they would have
more control over their own work (as they are not tied to the few
employers that own these popular products), and would also likely be
able to negotiate a larger portion of the compensation directly since
there would not be the overhead of salespeople and lawyers, etc. present in
current large software companies.

All Free Software users would benefit from the greater usage and
investment - GNU/Linux users could expect better hardware support for
example, as it becomes more commonplace.


## How to get there

Whilst the benefits of adopting FOSS are clear, the cases in Munich and
Hungary show that it will not be an easy path to greater adoption. We
cannot just rely on central government eventually carry out the adoption
as the few representatives are easily pressured by the beneficiaries of
the status quo, and are serving a much larger political platform.

Ultimately, I think the solution lies in applying "Linus's Law" to
politics itself. It is no coincidence that the successes in FOSS
adoption so far have been concentrated in local government - in specific
city councils and regional administrations, because it is easier for
engaged groups of citizens to have a direct effect in local government.

At a national level, it might be possible for a large software company
to pressure a few political leaders and policy makers, however, it is
much harder to do the same to a whole board of a dozen or more local
councillors (especially across the multitude of different counties and
regional administrations). There is "safety in numbers" as it becomes
unfeasible to pressure and manipulate large groups of
politically-engaged citizens.

It it is the responsibility of every citizen in a democratic society not
only to inform themselves and vote in national elections, but also to really partake in the
political system: by joining a political party, taking part in local and
regional elections, and ensuring that the democratic standards are
upheld both inside the party and in political institutions.

This should have support from across the political spectrum, as
guaranteeing the security and privacy of citizens' data whilst also
cutting out the middlemen of massive foreign corporations, shouldn't be
a controversial policy.

Efforts in local politics have already proven successful, such as the
LiMux project and others mentioned previously. So if you agree with the
points raised here I hope you will join whichever local political party
you agree most with, and help to bring about FOSS adoption by your
government.

## Summary

Greater FOSS adoption by our governments benefits the administrations
themselves by being free from repeated software licensing costs.

It would benefit us as citizens by helping to ensure that our data is kept secure and private from foreign
intelligence services. It would also benefit us as FOSS users by providing a
greater user base, ultimately leading to better hardware and software
support. 

Finally, it would benefit us as developers by providing greater
freedom in the possibilities of employment (since many different
employers could work on contracts for FOSS improvement), leading to a
better geographical distribution of opportunities and not being subject
to bad corporate practices such as non-compete agreements and
[anti-poaching agreements between large software companies](https://en.wikipedia.org/wiki/High-Tech_Employee_Antitrust_Litigation).

If you agree with the points in this post, please consider:
* Joining a local political party which you agree with, and supporting
  FOSS adoption
* Joining the [Public Money, Public Code campaign](https://publiccode.eu/)
* Supporting the [Free Software Foundation Europe (FSFE)](https://fsfe.org/) (or your [local Free Software Foundation](https://www.fsf.org/))
