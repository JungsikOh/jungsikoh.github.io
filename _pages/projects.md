---
layout: content
title: Projects
permalink: /projects/
sph: false
---

## Toy Projects

<style>
/* --- Projects page layout (self-contained) --- */
.projects-wrap { max-width: 1100px; margin: 0 auto; }
.top-links { margin: 0 0 2rem 0; }
.top-links li { margin: .25rem 0; }

.project { 
  display: grid; 
  grid-template-columns: minmax(260px, 1fr) 2fr; 
  gap: 2rem; 
  align-items: start; 
  margin: 3rem 0; 
}
.project__media img { 
  margin-top: 10px;
  width: 100%; 
  height: auto; 
  display: block; 
  border-radius: 10px; 
}
.project__media--thumbs { 
  display: grid; 
  grid-template-columns: repeat(3, 1fr); 
  gap: .75rem; 
}
.project__title { 
  margin: 0 0 .5rem 0; 
  font-size: clamp(1.12rem, 0.84rem + 0.84vw, 1.54rem); 
}
.project__desc { margin: 0 0 .5rem 0; line-height: 1.6; }

.dot-list { list-style: none; padding: 0; margin: .75rem 0 0 0; }
.dot-list li { position: relative; padding-left: 1.1rem; margin: .25rem 0; }
.dot-list li::before { content: "●"; position: absolute; left: 0; top: 0; line-height: 1; }

.project__links { margin-top: .75rem; }
.project__links a { text-decoration: underline; font-weight: 600; margin-right: 1rem; }

hr.project-sep { border: 0; height: 1px; background: rgba(0,0,0,.08); margin: 2.5rem 0; }

@media (max-width: 860px) {
  .project { grid-template-columns: 1fr; }
}
</style>

  <!-- Project 1: three thumbnails on the left, text on the right -->
  <section class="project">
    <div class="project__media">
      <img src="https://jungsikoh.github.io/assets/images/projects/riche.png" alt="Riche Vulkan render">
    </div>
    <div class="project__content">
      <h2 class="project__title">Riche Vulkan</h2>
      <p class="project__desc">
        Graphics renderer Project written in C++ using Vulkan, centered on the implementation of Batch Rendering. This renderer is aimed at improving real-time rendering speed through Batch-based draw calls.
      </p>
      <p class="project__links">
        <a href="https://youtu.be/x90SoRK9CGA?si=lGi7A22cP7v3_0XM">Video</a>
      </p>
    </div>
  </section>

  <hr class="project-sep" />

  <!-- Project 2: one big image on the left, text on the right -->
  <section class="project">
    <div class="project__media">
      <img src="https://jungsikoh.github.io/assets/images/projects/riley.png" alt="Riley Engine scene">
    </div>
    <div class="project__content">
      <h2 class="project__title">Riley Engine</h2>
      <p class="project__desc">
        Graphics renderer project written in C++ using DirectX 11, focusing on Tiled Deferred Lighting and various other advanced lighting techniques. It emphasizes lighting effects including Unreal Physically Based Rendering (PBR)
      </p>
      <p class="project__links">
        <a href="https://youtu.be/_Tj_7X7Es-4?si=wYUzIZCI2AITw2ly">Video</a>
      </p>
    </div>
  </section>

