---
layout: page
title: Geospatial Data Question Answering
description: NumGeoQA - Intelligent Question Answering for Geospatial Data
img: assets/img/3.jpg
importance: 2
category: work
giscus_comments: true
---

Geospatial data is employed to investigate factual occurrences on Earth, encompassing aspects such as temperature and the presence of objects. This data provides insights across a broad spectrum of domains, including forestry, agriculture, and socioeconomic factors. From space, numerous sensors are continuously acquiring information about the planetary surface and subsurface, offering a comprehensive global perspective. Simultaneously, human activities on the ground are constantly generating new data, providing a localized viewpoint. A significant portion of this collected data remains underutilized, representing a wealth of untapped information. The advancements in Artificial Intelligence (AI) present an opportunity to activate this dormant data, facilitating the connection of space-based and ground-based observations and enabling the discovery of patterns in our society and planet.

The rapid development of computer vision, particularly when combined with the capabilities of Large Language Models (LLMs), now enables the implementation of various open-world image tasks, such as identifying specific objects from visual bands. However, a substantial amount of data extends beyond the visible spectrum, and its semantic meaning transcends the information captured by physical equations and tailored models. High-level information is often distributed across different data modalities and varies across space and time, including correlations and causal relationships between observed phenomena. The independent analysis of computer vision alone cannot fully address such demands. LLMs, enhanced with tools and web-based knowledge, are making intelligent geospatial data analysis increasingly feasible. Furthermore, the emergence of automatic research agents is propelling this trend forward.

The facts we seek from geospatial data are predominantly presented as numerical answers, such as length, area, height, location, volume, and correlation coefficients. The accuracy of these answers is contingent upon the selection and availability of data, much of which remains stored and potentially unknown within data servers, especially as the volume of this data continues to grow. However, the current paradigm in Computer Vision (CV) and Natural Language Processing (NLP), which typically involves a specific "provide-data then ask" or non-provide approach, is not ideally suited for the development of Big Geospatial Data (BigGeoData). Unlike NLP and CV, geospatial data users often do not possess the data themselves, and the selection of appropriate data necessitates the expertise of data providers or scientists specializing in this field.

To address this critical gap, NumGeoQA is introduced as a novel, large-scale spatial-temporal based geospatial dataset specifically engineered to accelerate the development of advanced geospatial agents. NumGeoQA is distinguished by its high concentration of spatial-temporal based numerical answers, significantly surpassing the limited availability, extensibility, and temporal-spatial and factual information found in existing open vision and text datasets (Figure 1a). The dataset construction methodology incorporates the FAIR principles against established benchmarks (Figure 4) to ensure trustworthy evaluation, alongside ongoing extension, updating, and contributions from the community. The concept of a real geospatial agent is rooted in a data framework utilizing open-source data, where contributors only need to provide the location, time tags, and the facts they have observed or measured. The accuracy of the answers will improve with the updating of available data sources and the reasoning capabilities of the Agents.

The effectiveness of NumGeoQA is validated by demonstrating that models trained on it achieve substantial performance gains on several open-data-based geospatial reasoning benchmarks. This work furnishes a vital, openly accessible resource, tackling the urgent need for global, spatial-temporal, verifiable, and updating Geospatial Question Answering (QA) essential for propelling the next generation of Intelligent Geospatial Agent systems.


    ---
    layout: page
    title: project
    description: a project with a background image
    img: /assets/img/12.jpg
    ---

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/1.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/3.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/5.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Caption photos easily. On the left, a road goes through a tunnel. Middle, leaves artistically fall in a hipster photoshoot. Right, in another hipster photoshoot, a lumberjack grasps a handful of pine needles.
</div>
<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/5.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    This image can also have a caption. It's like magic.
</div>

You can also put regular text between your rows of images.
Say you wanted to write a little bit about your project before you posted the rest of the images.
You describe how you toiled, sweated, *bled* for your project, and then... you reveal its glory in the next row of images.


<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/6.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/11.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    You can also have artistically styled 2/3 + 1/3 images, like these.
</div>


The code is simple.
Just wrap your images with `<div class="col-sm">` and place them inside `<div class="row">` (read more about the <a href="https://getbootstrap.com/docs/4.4/layout/grid/">Bootstrap Grid</a> system).
To make images responsive, add `img-fluid` class to each; for rounded corners and shadows use `rounded` and `z-depth-1` classes.
Here's the code for the last row of images above:

{% raw %}
```html
<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/6.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/11.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
```
{% endraw %}
