---
layout: content
title:
sph: false
---


<!-- ────────── ❶ 유체 배경을 그릴 캔버스 ────────── -->
<canvas id="fluid-canvas"></canvas>

<!-- ────────── ❷ 스타일 ────────── -->
<style>
/* 페이지 전체(Fluid 캔버스가 뒤에 깔려야 하므로) */
html,body{margin:0;height:100%}

/* Fluid 캔버스: 화면을 꽉 채우고 콘텐츠 아래(z-index:-1)로 */
#fluid-canvas{
  position:fixed; inset:0;
  width:100%; height:100%;
  z-index:-1; display:block;
}

/* ── Hero 섹션 ─────────────────── */
.hero{
  height:85vh;                /* 화면 85 % 차지 */
  display:flex; flex-direction:column;
  justify-content:center; align-items:center;
  gap:1.8rem; text-align:center;
}

.hero__title{
  --chars: 10;
  width: 0ch;

  margin:0;
  font-size:clamp(2.5rem,6vw,5rem);
  font-weight:800;
  background:linear-gradient(90deg,#0d6efd,#6610f2);
  -webkit-background-clip:text;
  -webkit-text-fill-color:transparent;
  white-space:nowrap; overflow:hidden;
  border-right:.08em solid #666;
  animation:typing 2.2s steps(10, end) forwards,
           blink .75s step-end 2.8s infinite;
}

@keyframes typing{
  from{ width: 0ch; }
  to  { width: 10ch; }
}
@keyframes blink {50%{border-color:transparent}}

.hero__subtitle{
  margin:0 1rem;
  font-size:clamp(1.1rem,2.5vw,1.75rem);
  color:#eeeeee;
  max-width:60ch;
}
</style>

<!-- ────────── ❸ Hero 콘텐츠 ────────── -->
<div class="hero">
  <h1 class="hero__title">Jungsik Oh</h1>
  <p class="hero__subtitle">
    Computer Graphics &nbsp;•&nbsp; AI
  </p>
</div>

<!-- ────────── ❹ WebGL-Fluid 라이브러리 & 초기화 ────────── -->
<script src="https://cdn.jsdelivr.net/npm/webgl-fluid-enhanced@0.8.0/dist/webgl-fluid-enhanced.min.js"></script>
<script>
const canvas = document.getElementById('fluid-canvas');

webGLFluidEnhanced.simulation(canvas,{
  SIM_RESOLUTION:256,
  DYE_RESOLUTION:512,
  PRESSURE:0.8,
  SPLAT_RADIUS:0.004,
  BACK_COLOR:'#0a0a0a',              // 어두운 바탕
  COLOR_PALETTE:['#98BBEB','#f06595','#f8c555'] // 유체 색상
});

// 자동으로 물감이 퍼지도록 가벼운 스플랫 추가(선택)
setInterval(()=>webGLFluidEnhanced.splats(1),2200);
</script>