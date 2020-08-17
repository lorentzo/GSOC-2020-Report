# Google Summer of Code 2020 Report - Microfacet Based Normal Mapping

## Introduction

My first encounter with appleseed was in Summer 2019 when I was finishing my master theses at the University of Zagreb and applied for an internship at Fraunhofer ITWM. Then, I have solved my first issue and learned a bit about the appleseed rendering engine, code, and community. During my internship and later during my Ph.D. at Fraunhofer ITWM, I have been using appleseed quite a lot. As the plan was to use the appleseed further in my Ph.D., Petra Gospodnetic encouraged me to sign up for the GSoC with appleseed. My mentors Markus Rauhut and Hans Hagen agreed with this for which I want to thank them!

During the GSoC application period I was tackling with the [Importance sampling of OSL environment EDFs #701](https://github.com/appleseedhq/appleseed/issues/701). Also, I was looking into the offered project proposals and the existing issues and trying to figure out what will be useful both for appleseed, my Ph.D., and my general interests. I have decided to write a proposal for microfacet based normal mapping which was one of the issues [Investigate more robust normal mapping #2427](https://github.com/appleseedhq/appleseed/issues/2427) and which was interesting to me due to the possibility to learn more about the BRDF basis modification techniques for the spatial variation in modern rendering engines. 

In the end, I was accepted as a GSoC student! This allowed me to work on the microfacet based normal mapping implementation based on Schüßler et al. [1] for three months, mentored by Lars with a lot of help and support from François, Esteban and others from appleseed. Finally, I got paid by Google during all three months. Therefore, I want to thank them all!

## About Normal Mapping, How It Can Be Enhanced And My Contribution

My project was based on Schüßler et al. "Microfacet-based Normal Mapping for Robust Monte Carlo Path Tracing" [1]. This paper explains the problem with existing normal mapping techniques and offers a solution. First, a general discussion about the normal mapping will follow. Next, the solution for enhancing the normal mapping techniques will be discussed. Finally, project goals and contributions will be presented.

### About Normal Mapping

In general, surface spatial variation (e.g. bumps, dents, rust, etc.) on a flat geometrical surface can be created by varying BRDF parameters (such as roughness, albedo, etc.) or modifying basis on which the BRDF is evaluated. Normal mapping is one of the basis modification techniques. Instead of using geometrical normal, a normal sampled from a normal map is used. This method allows using simple geometry while having the appearance of complex spatial variation in the final render. It is important to note the word "appearance" was used because normal mapping is used for shading calculation and it results in highlights and shadows which would be present if the real geometry is used - but the real geometry is not present in the scene.

While it has good sides, normal mapping is an old technique and it causes various problems in the modern Monte Carlo rendering as discussed in Schüßler et al. [1, section 3]:
1. Non-symmetric BRDF due to the BRDF basis constructed using shading normals sampled from the normal map (Figure 1). The problem appears because the foreshortening factor for light direction (`wo`) must use normal sampled from the normal map (`ws`) instead of the geometric normal (`wg`) leading to the modified BRDF which is non-symmetric even if the original BRDF was symmetric.
2. Tilting of the positive hemisphere of the outgoing directions, due to the shading normal (`ws`) sampled from the normal map, causes inconsistencies with the geometric (`wg`) hemisphere. Due to the tilting, light can leak through the surface (Figure 2, left) or the BRDF becomes undefined for directions below the tilted hemisphere  (Figure 2, right). This effect is visible as black areas on the final render.
3. Violation of the energy conservation which becomes present because the modified foreshortening factor for the light direction can become arbitrary large for the light directions nearly perpendicular to the geometrical surface.
  
| ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/tilt1.png) | 
|:--:| 
| *Figure 1: Eye path (source: [1]).* |

| ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/hemisphere_tilt.png) | 
|:--:| 
| *Figure 2: Hemisphere tilt (source: [1]).* |

### Solution For Enhancement

The core idea of the paper is to instead of just replacing the geometrical normal with the normal sampled from a normal map for building the BRDF basis and the BRDF evaluation, the sampled normal is used for the construction of the microfacet-like surface model in a shading point. The resulting microfacet-like model is used for the BRDF evaluation. From now on, 'perturbed normal' will denote the normal sampled from a normal map as it is done in the paper [1].

Authors have proposed a model for the surface based on the microfacet theory. They have introduced two facets per shading point (Figure 3). One facet has a direction of the perturbed normal and it is called perturbed facet (`wp`). The other facet is used so that the average microfacet normal in a shading point equals to the geometric normal and it is called tangent facet (`wt`). The perturbed facet always has the BRDF set by user while constructing a material. The tangent facet BRDF can be either specular or same as the perturbed BRDF. For example, if a user, during material creation, decides on the `glossyBRDF` and a particular normal map then the perturbed facet will have the direction equal to the normal sampled from the normal map and the facet BRDF will be `glossyBRDF`.

| ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/profile.png) | 
|:--:| 
| *Figure 3: Microsurface profile (source: [1]).* |

Based on this model, authors have derived the multiple-scattering BRDF which is then evaluated per shading point. A solution for this BRDF was given by using the random walk with the arbitrary scattering orders which can be quite slow. Now the important simplification comes in. As mentioned, tangent facet BRDF can be the same as the perturbed facet BRDF or specular BRDF. It is important to note that the tangent facet has as little influence as possible (remember that it is used so that the average microfacet normal in a shading point equals to the geometric normal). Choosing the specular BRDF for the tangent facet, evaluation of the tangent facet is actually the evaluation of the perturbed facet BRDF with the outgoing/incoming directions reflected on the tangent facet. Finally, choosing the tangent facet BRDF to be specular, 2nd order scattering has an analytical solution. This analytical solution, according to the paper [1, section 7], it is the "best default model":
1. The authors have noted that the tangent facet with the specular BRDF removes most artifacts and produces results close to the classical normal mapping.
2. The analytical model can be evaluated fast without additional variance compared to the random walk model.

### My Contribution

Regarding the contribution, my plan was the following:
1. Investigate the paper [1] in detail. Focus on the model with the analytical solution and its elements.
2. Investigate the code given by the authors in Mitsuba.
3. Investigate the appleseed shading modules.
4. Implement the minimal working example for the `metalBRDF` in the appleseed.
5. Test the minimal working example.
6. Enhance the minimal working example and expand it to other BRDFs by creating a helper/wrapper class.
7. Enable the usage of the model in both appleseed built-in (`generic`) and OSL material.
8. Optional: investigate the arbitrary order scattering model from the paper [1], investigate the solution for this model without the random walk, and implement it as a helper class so it can be used for arbitrary BRDFs.
9. Create the showcase test scenes. 

My overall contribution at the end of the GSoC:
1. 2nd order scattering model with the specular tangent facet (analytical model) is implemented as `MicrofacetBRDFWrapper` class. This class performs the microfacet based normal mapping independent of specific BRDF.
3. Microfacet based normal mapping usage is enabled for both the appleseed built-in (`generic`) and the OSL material.
4. New test scenes, as a showcase for the microfacet based normal mapping, which use several BRDFs, area and point lights, different normal maps, and creation of the material using the appleseed built-in (`generic`) and the OSL material.

## Per Aspera: Coding Part

Before the coding period, one month was given for the community bonding. I have used that time for two main purposes. First, I was investigating the paper in more depth focusing on the analytical model that I was planning to implement. This investigation was accompanied by the examination of the implementation in Mitsuba that was provided by the authors. My goal was to understand the design decisions and implementation details on how the `BRDF::sample()`, `BRDF::evaluate()` and `BRDF::evaluate_pdf()` were modified. On the other side, I was exploring the appleseed and its rendering process. My goal was to understand the implementation details that would give me the foundations for my implementation. I was exploring shading modules, BRDFs, OSL. I was mostly focused on the `metalBRDF`. By the end of the bonding period, I have started the implementation phase.

During the first month of the coding period, I wanted to finish the minimal working example. I have decided to implement the microfacet based normal mapping for the `metalBRDF`. Starting enthusiasm due to finishing the minimal working example vanished quickly when the results were catastrophic. Most of the first month ended up in testing and figuring out the bugs. For that purpose, I was using different normal maps, lighting, camera positions, rendering log information, etc. One notable problem was figuring out the vector space (i.e. local or global) for shading calculation in Mitsuba and which space should I use for my implementation. For my implementation, I have decided on the global space. Also, another problem worth mentioning was due to notation differences between appleeseed, Mitsuba, and the paper which took me a quite amount of the time.

During the second month of the coding period, I was still performing a lot of tests. But then the significant progress came. One of the mistakes was that I was using too complex test scenes. Therefore, I have learned how to bake the normal maps in Blender. This allowed me to create normal maps from geometries that I have created. This was very useful because I had the real geometry at hand. Also, it was useful because I could create normal maps ranging from very simple to very complex ones. Next, I have compiled the Mitsuba with the code given by the authors. This was a very important step because I could render the same test scenes both in appleseed and Mitsuba with the same normal maps that I have created. This allowed me to compare the results of my work with the results from the authors. Also, I have implemented a color coding in the appleseed and the Mitsuba which colors the final render based on certain function calls. This helped me to compare the modules in my implementation with the modules in the Mitsuba implementation. During this time I have learned a lot about normal maps in general and scene creation in appleseed and Mitsuba. All of this helped me a lot to fix and improve my implementation. To satisfy the goal of using the microfacet based normal mapping for different BRDFs, I have extracted the microfacet based normal mapping code from the `metalBRDF`, enhanced it, and created a wrapper class which is independent of the particular BRDF. 

In the third and the final month of the coding period, I was baking more and more complex normal maps for test scenes. Also, I have started using area lights for testing the implementation. This has helped me to reveal some problems that had to be improved so I was working on that. I have managed to match the appleseed and Mitsuba renders and my results were becoming quite good. To enable usage of the microfacet normal mapping for different BRDFs I have created new BRDFs and closures which use desired BRDFs but wrapped with microfacet based normal mapping. Finally, I was wrapping up my work by opening the PR and finishing up showcase test scenes. The final PR contained both the complete code and test scenes. The reason for the one big PR is that the rendering process is dependent on all modules of my implementation.

During the whole community bonding and coding period, I had weekly meetings with Lars. I was keeping the community updated by writing daily TODOs, contributions, results, and summaries. For that purpose, I was using a dedicated Discord channel. Lars, François, Esteban, and others helped with useful information and support. Also, I was writing a daily journal to keep my thoughts organized.

## Ad Astra: Final Results

In the third month of the coding period, I had finally some nice results. A comparison with Mitsuba proved that. The overall goal of keeping the results as similar as possible to the classical normal mapping but removing most of the artifacts was met. As images will tell more than I can describe, a comparison of appleseed classical and microfacet based normal mapping will follow. Most of the normal maps that I have been using I have created in Blender. The process of creating normal maps was particularly interesting for me because it allowed my artistic side to explore and create. Also, it was important to create test scenes that would cover my contribution. Therefore, I have used `metalBRDF`, `glossyBRDF`, `plasticBRDF`, `sheenBRDF`, `lambertianBRDF`, and `blinnBRDF` in combination with point and area lights with different normal maps and two main material creation processes in appleseed: built-in (`generic`) material and OSL material. Although the material creation process is not changing the final render, it was important to show that both is possible. 

#### Metal BRDF, area light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/27original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/27.png)

#### Glossy BRDF, area light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/34original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/34.png)

#### Plastic BRDF, point light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/31original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/31.png)

#### Lambertian BRDF, point light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/32original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/32.png)

#### Blinn BRDF, point light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/33original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/33.png)

#### Sheen BRDF, point light
Classical normal mapping             |  Microfacet based normal mapping
:-------------------------:|:-------------------------:
![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/35original.png)  |  ![](https://github.com/lorentzo/GSOC-2020-Report/blob/master/Images/35.png)

The PR containing complete code and test scenes: [GSOC microfacet normal mapping #2886](https://github.com/appleseedhq/appleseed/pull/2886). In the time of submitting this report the PR was still open. 

## Looking In The Future

For future work, I would point out the multiple scattering model. Evaluation of the multiple scattering model in the paper [1] is solved using a random walk which is not the preferred way to do it in appleseed, therefore, more investigation should be done. Unfortunately, I hadn't had enough time to investigate and implement this model.  On the other hand, the authors concluded that the analytical 2nd order scattering model seems "the best default model". It removes most of the artifacts while producing results close to the classical normal mapping yet it can be evaluated quickly without additional variance compared to the random walk model.

## Conclusion

In the end, I can say that I have learned a lot. Firstly, I have learned about the appleseed code, rendering process, materials, and how is it like to contribute to a real rendering engine which will help me in further contributions to the appleseed. I have learned more about Mitsuba and Blender. Next, this is my first time implementing a model from a paper that will definitely make future paper implementations easier. Probably one of the most important things for me to learn was to “make everything as simplest as possible” and this is something I will always try to remember myself. I have also learned about setting up the experimental scenes and methods for testing. There is also a lot of things that I have learned about the interaction with others. Finally, I have learned about myself: how can I handle this kind of work, how to deal with priorities, how to deal with stress, etc. This wouldn’t be possible if I haven’t been chosen for a GSoC student. Therefore, I would like to thank the appleseed organization, TU Kaiserslautern, and Fraunhofer ITWM who made this possible for me! Last but not least, I want to thank my mentor Lars for time, patience, and useful discussions, also I want to thank François and Esteban who also found time for discussions or dropping by with words of support. 

## References:

[1] "Microfacet-based Normal Mapping for Robust Monte Carlo Path Tracing" Vincent Schüßler (KIT), Eric Heitz (Unity Technologies), Johannes Hanika (KIT) and Carsten Dachsbacher (KIT), ACM SIGGRAPH ASIA 2017
