---
layout: post
title: "mendelian"
titleen: "Introduction to Mendelian Randomization"
titlesp: "Introducción a aleatorización mendeliana"
date: 2014-08-17 15:22:20 +0200
comments: true
categories: [Statistics]
meta: false
---

<div class="English">
<h2>Introduction</h2>

<p>Mendelian Randomization is an approach to test for a <b>causal</b> effect from <b>observational</b> data in the presence of certain <b>confounding</b> factors.
 It uses the measured variation of genes (of known function) to <b>bound</b> the causal effect of a modifiable exposure (environment) on a phenotype (disease). The fundamental idea is that the genotypes are <b>randomly assigned</b> (due to recombination in meiosis under certain assumptions), and this allows them to be used as an <b>instrumental variable</b>.</p>

<!--more-->

<p>Here is the Directed Acyclic Graph (<b>DAG</b>) for the basic set-up. Here <b>G</b> is the observed genotype (i.e. the presence of a SNP), <b>X</b> is an environmental exposure, <b>U</b> is a (possible) unobserved confounder and <b>Y</b> is the phenotype (i.e. disease status). This will be explained in detail later.</p>


<img src="/images/simpmr.png" height="200">

<h2>Motivation</h2>

<p>Mendelian Randomization is a useful technique precisely because it allows for causal inference from <b>observational data</b>. <b>Randomised Control Trials</b> (RCTs) are the gold standard for causal inference, but is not always ethical or possible to carry out RCTs. For example, we cannot randomly assign a lifetime of heavy smoking (or non-smoking) to groups of individuals. This leads to a need to use observational data, however this requires many <b>assumptions</b>. </p>

<h2>Instrumental Variables (IVs) </h2>

<p>In the previous <b>DAG</b>, <b>G</b> is the <b>instrumental variable</b> (or instrument), because it affects <b>Y</b> only through <b>X</b> (that is, exclusively, no other path may exist). Therefore, under certain <b>assumptions</b> (to be explained later), if <b>G</b> is correlated with <b>Y</b> then we can infer the edge <b>X -> Y </b> (note that <b>X</b> must be correlated with <b>G</b>).</p>

<p>Now we will consider an example application of instrumental variables, for a hypothetical investigation of the effect of smoking on lung cancer.</p>

<img src="/images/smoking.png" height="200">

<p>If the <b>assumptions</b> of the instrumental variable method are met, then if we observe that an increase in tax leads to a reduction in lung cancer, then one can infer that smoking is a "cause" of lung cancer (though perhaps indirectly, i.e. it's not smoking directly that causes cancer but the build-up of tar in the lungs, etc.).</p>


<h2>Assumptions of the Instrumental Variables method</h2>
<p>Assumptions are necessary in all areas of statistics, however it is important to know what assumptions we are making and whether they hold true.</p>

<h3>Z -> X must exist</h3>

<p> We must know <b>a priori</b> that the causal direction is <b>Z -> X</b> and not <b>X -> Z</b>. This is what makes the causal structure unique and <b>identifiable</b>. Note this does not mean that <b>Z</b> has to be the "true" cause of <b>X</b>. For example, if <b>Z</b> is a SNP, we can choose a SNP in linkage disequilibrium with <b>Z</b>, so long as it is independent of all of the other variables, but still correlated with <b>X</b>. Note the more correlated that <b>Z</b> is with <b>X</b>, the better the power will be (i.e. less data required for significant results), if it is not correlated, it cannot be used as an instrumental variable.</p>

<img src="/images/snpld.png" height="200">

<h3>Z must be independent of U </h3>

<p> No factor can affect both the instrument and the effects. For example, there cannot be a factor that causes both higher tobacco taxes and less cancer (e.g. if we were comparing rates in different cancers between countries, national health awareness could be such a factor).</p>

<img src="/images/noug.png" height="200">


<h3>No Z -> Y</h3>

<p><b>Z</b> cannot directly affect <b>Y</b> (or indirectly, except through <b>X</b>). I.e. there cannot exist any other mechanisms through which <b>Z</b> affects <b>Y</b> (i.e. high tobacco tax increases substance abuse, leading to higher rates of cancer).</p>

<img src="/images/nozy.png" height="200">

<h3>Faithfulness</h3>

<p>Faithfulness is the assumption that the true underlying DAG will manifest itself in the <b>observed data</b> - that is, that the causal effects will not <b>cancel out</b>. This is a reasonable assumption, because the contrary would require very specific parameters.</p>

<p>However, note that if relations are deterministic, the implied conditional independencies of the DAG do not hold true, and the assumption of faithfulness is violated. But we will not concern ourselves with this.</p>

<p> Also note that, in practice, the <b>sample size</b> is very important in testing the significance of the correlations/independencies in the data.</p>

<h3>Summary of assumptions </h3>

<p>This DAG summarizes the necessary assumptions for the use of instrumental variables:</p>

<img src="/images/fulldagass.png" height="200">

<h2>Mendelian Randomization</h2>

<p>The method of Mendelian Randomization was first introduced in: <i>Apolipoprotein E isoforms, serum cholesterol, and cancer.</i>, Katan MB (1986).</p>

<p> At the time, epidemiologists were wondering whether low serum cholesterol levels increase the risk of cancer. They were known to be correlated, but it could be that latent tumors caused the lower cholesterol levels (i.e. reverse causation), or that both cancer risk and cholesterol levels were affected by another factor, such as diet (i.e. confounding).</p>

<p>However, Katan noticed that patients with Abetalipoproteinemia (a genetic disease that leads to the inability to absorb cholesterol), did not appear predisposed to cancer, despite the predisposition to lower levels of serum cholesterol.</p>

<p>This led Katan to the idea of finding a large group of individuals genetically predisposed to lower cholesterol levels. These individuals are assumed to be the same with respect to other possible confounders (social class, etc.) and so the presence of the cholesterol-affecting allele can be used as an instrumental variable - this is <b>Mendelian Randomization</b>.</p>

<p> The Apolipoprotein E (<b>ApoE</b>) gene was known to affect serum cholesterol, with the <b>ApoE2</b> variant being associated with lower levels. Many individuals carry ApoE2 variant and so have <b>lower serum cholesterol levels</b> from birth.<p>

<p> Since genes are <b>randomly assigned</b> during meiosis (due to recombination), ApoE2 carriers should not be different from ApoE carriers in any other way (diet, etc.), so there is no confounding via the genome - note these are <b>assumptions</b>. It is this which allows the genotype to be used as an instrumental variable in this way (because it is akin to an intervention in a Randomised Control Trial).</p>

<p>Therefore if low serum cholesterol is really causal for cancer risk, the cancer patients should have more ApoE2 alleles than the controls - if not then the levels would be similar in both groups.</p>

<p>Katan only provided the suggestion, but the method has since been used for many different analyses with some success, such as the link between blood pressure and stroke risk. However, some conclusions have later been disproved by Randomised Control Trials. To understand why, we must consider the <b>biological assumptions</b>.</p>


<h2>Biological Assumptions</h2>
<h3>Panmixia</h3>
<p>Recall the assumption that the genotype is randomly assigned - this implies <b>panmixia</b>. That is, there is <b>no selective breeding</b> (so random mating). This implies that all recombination of alleles is possible.</p>

<p>In our DAG, this means that <b>G</b> is not influenced by <b>Y</b> (or any other variables). However, this assumption is not always entirely accurate, as demonstrated by <b>Population Stratification</b>

<h3>Population Stratification</h3>
<p>Population Stratification is the <b>systematic difference</b> in allele frequencies between subpopulations, due to ancestry. For example, physical separation leads to <b>non-random mating</b>. This then leads to different <b>genetic drift</b> in different subpopulations (i.e. changes in allele frequency over time due to repeated random sampling).</p>

<p>This means that the genotype is <b>not</b> randomly assigned when considered across different sub-populations, a good example of this is with the difference in rates of lactose intolerance between Northern Europe and Asia.</p>

<h3>Canalization</h3>
<p>Canalization refers to the variation in the <b>robustness of phenotypes</b> to changes in the genotype and environment.</p>

<p>The classic example is Waddington's Drosophilia experiment. In the experiment, Drosophilia pupae were exposed to heat shock (i.e. a rapid increase in temperature). Eventually a Cross-veinless phenotype (no cross-veins in wings) was produced. However, by then selectively breeding the resulting flies for this phenotype, it eventually appears without heat shock.</p>

<p>This led to theory of organisms rolling downhill in to "canals" of the <b>epigenetic landscape</b> with development, becoming more robust to variation of the environment. We can think of it like an optimization problem which settles in local minima.</p>

<p> The exact biological/molecular mechanisms of canalization are still unknown. With regards to Mendelian Randomization, it can act as a <b>confounder</b> between the genotype, environment and phenotype.</p>

<h3>No Pleiotropy</h3>
<p>Pleiotropy is the phenomena whereby one gene can affect many (even seemingly unrelated) phenotypes. Mendelian Randomisation makes the assumption of <b>no pleiotropy</b>.</p>

<p> In this case, this means that we assume the genotype is only influencing the phenotype via the considered exposure. I.e. ApoE2 <b>only</b> affects serum cholesterol levels, and cannot affect cancer risk by other, unobserved means.</p>

<p> This is a big assumption, and <b>prior knowledge</b> is necessary. If possible, using multiple, independent SNPs which act through the same path, can help to alleviate this issue, because, if they are all consistent then it is unlikely that they would all have other pathways causing the same change in phenotype. But note that they must be independent, and so cannot be in Linkage Disequilibrium.</p>


<h3>The real DAG?</h3>
Considering the above points, perhaps the true underlying DAG looks more like this:

<img src="/images/fulldagbio.png" height="200">

<h2>Conclusion</h2>
<p><b>Instrumental variables</b> are a method to infer causal relations from <b>observational data</b>, given certain <b>assumptions</b>.This method is applied in Genetic Epidemiology with <b>Mendelian Randomisation</b>.</p>
<p>This approach has had some success, but the underlying biology still poses some problems with regards to the necessary assumptions. This leaves us with the following questions:
<ul>
<li> Can we improve the <b>robustness</b> of the inference with more measurements of intermediate phenotypes? (such as gene methylation, RNAseq, proteomics measurements, etc.)?<br>

Some work has been done on this under the name of  multi-step Mendelian Randomisation. </li>
<br>

<li>Can we improve identification of appropriate <b>instruments</b>? (e.g. as whole genome sequencing makes it easier to identify population stratification)</li>

</ul>

</p>
</div>

<!--end-->
<div class="Spanish">
<h2> Introducción </h2>


<p>La aleatorización mendeliana es un método para confirmar un efecto <b>causal</b> de datos observacionales, posiblemente en la presencia de <b>factores de confusión</b>.</p>

<p>El método utiliza la discrepancia medida de genes (de funciones ya sabidas) para limitar la influencia causal de una exposición (por ej. el ambiente) en un fenotípo (por ej. una enfermedad). La idea fundamental es que los genotípos son asignados <b>aleatoriamente</b> (por la recombinación durante la meiosis), y esto permita que se puede ser utilizados como <b>variables instrumentales</b>.</p> 
<!--more-->
<p>Aquí hay el grafo acíclico dirigido (<b>DAG</b> en inglés) de la idea fundamental. <b>G</b> es el genotipo observado (o sea, la presencia de un SNP), <b>X</b> es una exposición ambiente, <b>U</b> es un (posible) factor de confusión escondido y <b>Y</b> es el fenotipo (o sea, el estado de la enfermedad). Se explica esto en detalle más adelante. </p>

<img src="/images/simpmr.png" height="200">

<h2> Motivación </h2>

<p>La aleatorización mendeliana es una técnica útil porque permita la inferencia causal de <b>datos observacionales</b>. <b>Pruebas controladas aleatorias</b> son el patrón para hacer la inferencia causal, pero no siempre está posible o etíca hacerlas. Por ejemplo, no podemos asignar aleatoriamente una vida de fumar (o no fumar) a personas. Este problema significa que necesitamos usar los datos observacionales (que podemos colectar), sin embargo, hacer esto requiere muchas <b>suposiciones</b>.</p>

<h2>Variables Instrumentales (IVs) </h2>
<p>En el <b>DAG</b> anterior, <b>G</b> es la variable instrumental (o sea, el instrumento), porque afecta <b>Y</b> sólo por su influencia en <b>X</b> (o sea, exclusivamente - nada otros caminos existen). Por lo tanto, bajo <b>suposiciones</b> específicas  (que voy a ser explicadas más adelante), si <b>G</b> está en correlación con <b>Y</b>, podemos inferir que la arista <b>X -> Y</b> debe existir (nota que <b>X</b> debe estar en correlación con <b>G</b> para ser usado como variable instrumental.)</p>

<p>Ahora consideramos un ejemplo de la aplicación de las variables instrumentales, en una investigación hipotética de la relación entre el fumar y el cancer de pulmón.</p>


<img src="/images/smoking.png" height="200">

<p>Asumiendo que las <b>suposiciones</b> del método son correctas, si observamos que an aumento del impuesto causa una reducción en la frecuencia de cáncer de pulmón, podríamos inferir que fumar es una "causa" de cáncer de pulmón (pero quizá indirectamente, o sea, no sería fumar que causa el cáncer directamente sino la acumulación del alquitrán en los pulmones, etc.)</p>


<h2>Suposiciones del método de variables instrumentales</h2>

<p>Suposiciones son necesarias en todos las partes de la estadística, sin embargo, es muy importante saber cuáles suposiciones necesitamos y si son correctas.</p>

<h3> Z -> X debe existir</h3>
<p>Debemos saber <b>a priori</b> que la dirección causal es <b>Z -> X</b> y no <b>X -> Z</b>. Es por eso que podemos identificar la estructura causal <b>únicamente</b>.Nota que eso no significa que <b>Z</b> debe ser la causa "verdadera" de <b>X</b>. Por ejemplo, si <b>Z</b> es un SNP (Polimorfismo de nucleótido simple - o sea, una mutación en una sola base), podemos también elegir un SNP que está en desequilibrio de ligamiento (o sea, son heredados juntos) con <b>Z</b>, con tal de que sea independiente de todas las otras variables, pero todavía esté en correlación con <b>X</b>. Nota que cuánto más que <b>Z</b> está en correlación con <b>X</b>, mejor será el poder estadístico (o sea, necesitaríamos menos datos para resultados significados), y si <b>Z</b>b> no está en correlación con <b>X</b>, no se puede ser usado como una variable instrumental.</p>

<img src="/images/snpld.png" height="200">

<h3>Z debe ser independiente de U</h3>

<p> No se permite que nada factor afecten ambos el instrumento y los efectos. Por ejemplo, no puede ser que haya un factor que causa ambos un aumento en los impuestos de tabaco y indices bajados de cáncer (por ejemplo, si comparamos los indices de un tipo de cáncer entre países diferentes, la conciencia salud nacional puede ser un factor así).</p> 

<img src="/images/noug.png" height="200">


<h3>No Z -> Y</h3>

<p>No se permite que <b>Z</b> afecte <b>Y</b> directamente (o indirectamente, excepto a través de <b>X</b>). O sea, no deben existir nada más mecanismos, a través de que <b>Z</b> afecte <b>Y</b> (por ejemplo, si el aumento en el impuesto de tabaco causara un aumento en los indices del abuso de sustancias, y por eso, un aumento en los indices de cáncer).</p>

<img src="/images/nozy.png" height="200">

<h3>Condición de Fidelidad</h3>

<p>La condición de Fidelidad es la suposición que la verdadero DAG se manifestará en los <b>datos observados</b> - o sea, los efectos causales <b>no se neutralizarán</b>. Esta es una suposición creíble, porque lo contrario necesitaría parámetros muy específicos.</p>

<p>Sin embargo, nota que si las relaciones son deterministas, las independencias condicionales que el DAG supone no más son correctas, y la condición de Fidelidad es violada. Pero ahorita eso no nos interesa.</p>

<p>Además, nota que en practica, el <b>tamaño de la muestra</b> es muy importante para probar la significación de las correlaciones estadísticas y independencias en los datos.</p> 
    
<h3>Resumen de las suposiciones</h3>

<p>Este DAG muestra las suposiciones necesarias para usar el método de variables instrumentales:</p>

<img src="/images/fulldagass.png" height="200">

<h2>Aleatorización mendeliana</h2>

<p>El método de aleatorización mendeliana fue presentado primeramente en <i>Apolipoprotein E isoforms, serum cholesterol, and cancer.</i>, Katan MB (1986).</p>

<p>En ese tiempo, los epidemiólogos se preguntaban si niveles bajos de colesterol en suero podrían aumentar el riesgo de cáncer. Se sabía que estaban en correlación, pero puede ser que tumores latentes causaran los niveles más bajos de colesterol (o sea, causalidad inversa), o que ambos el riesgo de cáncer y los niveles de colesterol eran afectados por otro factor, como la dieta (o sea, hay factores de confusión). </p>

<p>Sin embargo, Katan notó que los pacientes de Abetalipoproteinemia (una enfermedad genética que causa la incapacidad para absorber colesterol) no se parecen predispuestos al cáncer, a pesar de la predisposición a niveles más bajos de colesterol en suero.</p>

<p>Este hecho condujo Katan a la idea de encontrar un grupo grande de individuos que son predispuestos genéticamente a niveles más bajos de colesterol. Se asume que estes individuos son iguales con respecto a todos los factores posibles de confusión (por ej. clase social, etc.) así que la presencia del alelo que afecta el nivel de colesterol puede ser usado como una variable instrumental - esta idea es la aleatorización mendeliana.</p>

<p>El gen Apolipoprotein E (<b>ApoE</b>) era conocido afectar los niveles de colesterol en suero, la variante <b>ApoE2</b> vinculada con niveles más bajos. Muchos individuos tienen la variante ApoE2 y por eso tienen <b>niveles más bajos de colesterol en suero</b> desde el nacimiento.</p>

<p>Porque los genes son <b>asignados aleatoriamente</b> durante la meiosis (por la recombinación), los portadores del alelo ApoE2 no debería ser diferente de los portadores del alelo ApoE en nada otras formas (la dieta, etc.), por eso no hay nada confusión por el genoma - pero, nota que esas son <b>suposiciones</b>. Es por eso que se permite usar el genotipo como una variable instrumental (porque es como una intervención en una prueba controlada aleatoria).</p>

<p>Por eso, si niveles bajos de colesterol en suero causan un aumento en el riesgo de cáncer, los pacientes de cáncer deberían tener una frecuencia del alelo ApoE2 más alta que el grupo de control. Si la relación no es causal, los niveles serían aproximadamente iguales en ambos grupos.</p>

<p>Katan sólo dio la sugerencia, pero el método ha sido usado en muchos análisis con unos éxitos, como la relación entre la presión arterial y el riesgo de derrames cerebrales. Sin embargo, unas conclusiones han sido incorrectas (cuando han sido probadas por pruebas controladas aleatorias). Para entender la razón, tenemos que pensar en las <b>suposiciones biológicas</b>.</p>

<h2>Suposiciones biológicas</h2>
<h3>Panmixia</h3>
<p>Recuerda la suposición que el genotipo es asignado aleatoriamente - esto supone <b>panmixia</b>. Panmixia significa que no hay <b>reproducción selectiva</b> (o sea, la reproducción es aleatoria). Esto significa que todas recombinaciones de los alelos son posibles.</p>

<p>En nuestro DAG, esto significa que <b>G</b> no es influido por <b>Y</b> (o nada más variables). Sin embargo, esta suposición no es siempre verdadera, como demuestra la fenómeno de la <b>estratificación genética</b>.</p>

<h3>Estratificación genética</h3>

<p>La estratificación genética es la discrepancia sistemática en la frecuencia de los alelos entre subpoblaciones diferentes, por sus ascendencias. Por ejemplo, la separación física da lugar a <b>reproducción no aleatoria</b>. Esto da lugar a la <b>deriva genética</b> en las subpoblaciones diferentes (o sea, cambios en la frecuencia de los alelos con el tiempo por muestreo aleatorio repetido).</p>  

<p>Esto significa que el genotipo <b>no</b> es asignado aleatoriamente cuando es considerado entre las subpoblaciones diferentes. Un buen ejemplo de este fenómeno es la discrepancia en los indices de la intolerancia a lactosa entre el norte de Europa y Asia.</p>

<h3>Canalización</h3>
<p>Canalización se refiere al cambio en <b>la robustez de los fenotipos</b> a cambios en el genotipo y el ambiente.</p>

<p>El ejemplo clásico es el experimento de Waddington con pupas de drosophila. En este experimento, las pupas de drosophila fueron expuestos a choque térmico (o sea, un aumento rápido en la temperatura). Con el tiempo, un fenotipo con anormalidades apareció. Sin embargo, por criar selectivamente estas moscas que tuvo este fenotipo, con el tiempo el fenotipo aparece sin exposición a choque térmico.</p>

<p>Este experimento da lugar a la teoría de los organismos rodando abajo en "canales" del <b>paisaje epigenético</b> durante el desarrollo, se haciendo más resistente contra cambios en el ambiente. Se puede ser considerado como un problema de optimización, en que la solución queda en mínimos locales.</p>

<p>Los mecanismos precisos de canalización aún no son sabidos. Con respecto a la aleatorización mendeliana, puede ser un <b>factor de confusión</b> entre el genotipo, el ambiente y el fenotipo.</p>

<h3>No Pleiotropía</h3>

<p>La pleiotropía es el fenómeno por el cual un solo gen puede afectar muchos fenotipos. La aleatorización mendeliana requiere la suposición que no hay nada pleiotropía. </p>

<p>En este caso, signifíca que tenemos que suponer que el fenotipo sólo influye el fenotipo a través de la exposición considerado (por ej. ApoE2 sólo afecta los niveles de colesterol en suero, y no puede afectar el riesgo de cáncer por otros medias desconocidos).</p>

<p>Esta es una gran suposición, y <b>conocimiento previo</b> es necesario justificarla. Si es posible, usar SNPs independientes múltiples que actúan por el mismo camino puede ayudar mitigar este problema, porque si todos son consecuentes, es muy improbable que todos tendrían otros caminos para causar el mismo cambio en fenotipo. Pero nota que los SNPs deben ser independientes, y por eso, no pueden estar en desequilibrio de ligamiento.</p>

<h3>¿El DAG verdadero?</h3>
<p>Considerando los puntos anteriores, tal vez el grafo real parezca así: </p>

<img src="/images/fulldagbio.png" height="200">

<h2>Conclusión</h2>
<p>Las <b>variables instrumentales</b> son un método para inferir relaciones causales de <b>datos observacionales</b>, bajo unas <b>suposiciones</b>. Este método es aplicado en la epidemiología genética a través de la <b>aleatorización mendeliana</b>.</p>


<p>Este método ha tenido éxito, pero la biología subyacente todavía plantea unos problemas con respecto a las suposiciones necesarias. Esto nos deja las preguntas siguientes:

<ul>
    <li>¿Podemos mejorar la <b>robustez</b> de la inferencia con más medidas de fenotipos intermedios (como la metilación genética, RNAseq, medidas de la proteomíca, etc.)?
<br>
Algún trabajo ha sido hecho en este problema bajo el nombre de "multi-step Mendelian Randomization" (aleatorización mendeliana por etapas múltiples). </li>
<br>
    <li>¿Podemos mejorar la identificación de <b>instrumentos</b> apropriados? (por ej. la posibilidad de la secuenciación del genoma entero ha hecho más fácil identificar la estratificación genética)</li>
</ul>
</p>

</div>

