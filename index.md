---
layout: content
title: Jungsik Oh
sph: false
---

<!-- ───── Hero Section (Fullscreen) ───── -->
<style>
/* ① 배경 + 중앙 정렬 */
.hero{
  height:85vh;
  display:flex; flex-direction:column;
  justify-content:center; align-items:center;
  gap:1.8rem; text-align:center;
  background:
    radial-gradient(circle at 30% 30%,rgba(13,110,253,.25),transparent 60%),
    radial-gradient(circle at 70% 70%,rgba(13,110,253,.15),transparent 60%);
  background-color:#fdfdfd;
  overflow:hidden;
}

/* ② 타이핑 + 그라디언트 텍스트 */
.hero__title{
  font-size:clamp(2.5rem,6vw,5rem);
  font-weight:800; margin:0; line-height:1.2;
  background:linear-gradient(90deg,#0d6efd,#6610f2);
  -webkit-background-clip:text; -webkit-text-fill-color:transparent;
  white-space:nowrap; overflow:hidden;
  border-right:.08em solid #666;
  animation:typing 3.2s steps(30,end) 1,
           blink .75s step-end infinite;
}

@keyframes typing{from{width:0}to{width:100%}}
@keyframes blink {50%{border-color:transparent}}

/* ③ 부제목 */
.hero__subtitle{
  font-size:clamp(1.1rem,2.5vw,1.75rem);
  margin:0 1rem; color:#555; max-width:60ch;
}
</style>

<div class="hero">
  <h1 class="hero__title">JUNGSIK OH</h1>
  <p class="hero__subtitle">
    Computer Graphics  •  AI
  </p>
</div>
<!-- ─────────────────────────────────────── -->

