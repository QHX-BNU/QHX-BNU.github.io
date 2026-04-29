<div align="center">

<!-- HOLOGRAPHIC HEADER -->
<svg width="100%" height="200" viewBox="0 0 900 200" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <linearGradient id="holoGrad" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" stop-color="#4fc3f7" stop-opacity="0.3"/>
      <stop offset="20%" stop-color="#81d4fa" stop-opacity="0.9"/>
      <stop offset="50%" stop-color="#e1f5fe" stop-opacity="1"/>
      <stop offset="80%" stop-color="#81d4fa" stop-opacity="0.9"/>
      <stop offset="100%" stop-color="#4fc3f7" stop-opacity="0.3"/>
    </linearGradient>
    <linearGradient id="subGrad" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" stop-color="#0288d1" stop-opacity="0.4"/>
      <stop offset="50%" stop-color="#4fc3f7" stop-opacity="0.8"/>
      <stop offset="100%" stop-color="#0288d1" stop-opacity="0.4"/>
    </linearGradient>
    <filter id="glow">
      <feGaussianBlur stdDeviation="3" result="blur"/>
      <feMerge>
        <feMergeNode in="blur"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>
    <filter id="glowStrong">
      <feGaussianBlur stdDeviation="5" result="blur"/>
      <feMerge>
        <feMergeNode in="blur"/>
        <feMergeNode in="blur"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>
    <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
      <path d="M 40 0 L 0 0 0 40" fill="none" stroke="#4fc3f7" stroke-opacity="0.06" stroke-width="0.5"/>
    </pattern>
    <pattern id="scanlines" width="4" height="4" patternUnits="userSpaceOnUse">
      <rect width="4" height="2" fill="#0a1628" fill-opacity="0.3"/>
    </pattern>
  </defs>

  <!-- Background -->
  <rect width="900" height="200" fill="#0a1628" rx="12"/>
  <rect width="900" height="200" fill="url(#grid)" rx="12"/>

  <!-- Border glow -->
  <rect x="2" y="2" width="896" height="196" fill="none" stroke="#4fc3f7" stroke-opacity="0.2" stroke-width="0.5" rx="11"/>
  <rect x="4" y="4" width="892" height="192" fill="none" stroke="#4fc3f7" stroke-opacity="0.08" stroke-width="1" rx="10"/>

  <!-- Corner accents -->
  <path d="M20 20 L60 20 M20 20 L20 60" stroke="#4fc3f7" stroke-opacity="0.5" stroke-width="1.5" fill="none"/>
  <path d="M880 20 L840 20 M880 20 L880 60" stroke="#4fc3f7" stroke-opacity="0.5" stroke-width="1.5" fill="none"/>
  <path d="M20 180 L60 180 M20 180 L20 140" stroke="#4fc3f7" stroke-opacity="0.5" stroke-width="1.5" fill="none"/>
  <path d="M880 180 L840 180 M880 180 L880 140" stroke="#4fc3f7" stroke-opacity="0.5" stroke-width="1.5" fill="none"/>

  <!-- Horizontal scan line -->
  <line x1="0" y1="100" x2="900" y2="100" stroke="#4fc3f7" stroke-opacity="0.08" stroke-width="0.5"/>
  <line x1="0" y1="60" x2="900" y2="60" stroke="#4fc3f7" stroke-opacity="0.04" stroke-width="0.3"/>
  <line x1="0" y1="140" x2="900" y2="140" stroke="#4fc3f7" stroke-opacity="0.04" stroke-width="0.3"/>

  <!-- Data dots - left -->
  <circle cx="70" cy="30" r="1.5" fill="#4fc3f7" opacity="0.6"/>
  <circle cx="85" cy="30" r="1" fill="#4fc3f7" opacity="0.3"/>
  <circle cx="70" cy="170" r="1.5" fill="#4fc3f7" opacity="0.6"/>
  <circle cx="85" cy="170" r="1" fill="#4fc3f7" opacity="0.3"/>

  <!-- Data dots - right -->
  <circle cx="830" cy="30" r="1.5" fill="#4fc3f7" opacity="0.6"/>
  <circle cx="815" cy="30" r="1" fill="#4fc3f7" opacity="0.3"/>
  <circle cx="830" cy="170" r="1.5" fill="#4fc3f7" opacity="0.6"/>
  <circle cx="815" cy="170" r="1" fill="#4fc3f7" opacity="0.3"/>

  <!-- Top-left holographic label -->
  <rect x="80" y="18" width="70" height="14" rx="2" fill="#4fc3f7" fill-opacity="0.08" stroke="#4fc3f7" stroke-opacity="0.3" stroke-width="0.5"/>
  <text x="115" y="29" text-anchor="middle" fill="#4fc3f7" font-size="7" font-family="monospace" opacity="0.7">SYS::ONLINE</text>

  <!-- Top-right status -->
  <rect x="740" y="18" width="80" height="14" rx="2" fill="#4fc3f7" fill-opacity="0.08" stroke="#4fc3f7" stroke-opacity="0.3" stroke-width="0.5"/>
  <text x="780" y="29" text-anchor="middle" fill="#4fc3f7" font-size="7" font-family="monospace" opacity="0.7">NODE::ACTIVE</text>

  <!-- Main name -->
  <text x="450" y="105" text-anchor="middle" fill="url(#holoGrad)" font-size="48" font-family="Arial, sans-serif" font-weight="bold" filter="url(#glowStrong)" letter-spacing="12">阙 辉 兴</text>

  <!-- Underline accent -->
  <line x1="320" y1="125" x2="580" y2="125" stroke="#4fc3f7" stroke-opacity="0.4" stroke-width="0.5"/>
  <line x1="360" y1="128" x2="540" y2="128" stroke="#4fc3f7" stroke-opacity="0.2" stroke-width="0.3"/>

  <!-- Subtitle -->
  <text x="450" y="155" text-anchor="middle" fill="url(#subGrad)" font-size="13" font-family="monospace" letter-spacing="4" filter="url(#glow)">UNIVERSITY OF SCIENCE AND TECHNOLOGY OF CHINA</text>
  <text x="450" y="175" text-anchor="middle" fill="#4fc3f7" font-size="11" font-family="monospace" letter-spacing="3" opacity="0.6">COMPUTER SCIENCE</text>
</svg>

</div>

<br>

<!-- STATS BAR -->
<div align="center">

![Python](https://img.shields.io/badge/Python-4fc3f7?style=for-the-badge&logo=python&logoColor=white&labelColor=0a1628)
![PyTorch](https://img.shields.io/badge/PyTorch-4fc3f7?style=for-the-badge&logo=pytorch&logoColor=white&labelColor=0a1628)
![ACL 2026](https://img.shields.io/badge/ACL_2026-4fc3f7?style=for-the-badge&labelColor=0a1628)
![USTC](https://img.shields.io/badge/USTC-4fc3f7?style=for-the-badge&labelColor=0a1628)

</div>

<br>

<!-- ============================================ -->
<!-- RESEARCH SECTION -->
<!-- ============================================ -->

<table>
<tr><td>

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ◆  RESEARCH                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

</td></tr>
</table>

<table>
<tr>
<td width="72px" align="center" valign="top">
  <br>
  <img src="https://img.shields.io/badge/ACL_2026-4fc3f7?style=flat-square&labelColor=0a1628" />
</td>
<td>

### One LLM Does Not Simulate All Students
**Ability-Aware Student Simulation via Cognitive Diagnosis Guided LLM Assignment**

*Large Language Models (LLMs) have become integral to personalized education systems. We demonstrate that a "one-size-fits-all" approach induces systematic **ability-dependent bias**, and propose an ability-aware framework that dynamically matches students with appropriate LLM backbones through cognitive alignment.*

<br>

[![Paper](https://img.shields.io/badge/Paper-OpenReview-4fc3f7?style=flat-square&logo=readthedocs&logoColor=white&labelColor=0a1628)](https://openreview.net/forum?id=IfIRg71R61)
[![Code](https://img.shields.io/badge/Code-GitHub-4fc3f7?style=flat-square&logo=github&logoColor=white&labelColor=0a1628)](https://github.com/QHX-BNU/Ability-Aware-Student-Simulation)

</td>
</tr>
</table>

<br>

<!-- ============================================ -->
<!-- PROJECTS SECTION -->
<!-- ============================================ -->

<table>
<tr><td>

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ◆  PROJECTS                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

</td></tr>
</table>

<table>
<tr>
<td width="50%">

<table>
<tr>
<td>

```
  ╔══════════════════════════════════╗
  ║  ABILITY-AWARE STUDENT SIM      ║
  ╚══════════════════════════════════╝
```

![Python](https://img.shields.io/badge/Python-4fc3f7?style=flat-square&logo=python&logoColor=white&labelColor=0a1628)
⭐ 1

Official implementation of our ACL 2026 paper. An end-to-end pipeline integrating Neural Cognitive Diagnosis with LLM-based student simulation.

[![Repo](https://img.shields.io/badge/Repository-4fc3f7?style=flat-square&labelColor=0a1628)](https://github.com/QHX-BNU/Ability-Aware-Student-Simulation)

</td>
</tr>
</table>

</td>
<td width="50%">

<table>
<tr>
<td>

```
  ╔══════════════════════════════════╗
  ║  EduData                        ║
  ╚══════════════════════════════════╝
```

![Star](https://img.shields.io/badge/★_1-4fc3f7?style=flat-square&logo=github&logoColor=white&labelColor=0a1628)
![Fork](https://img.shields.io/badge/⑂_1-4fc3f7?style=flat-square&logo=git&logoColor=white&labelColor=0a1628)

Educational datasets collection with detailed descriptions for quick understanding and research use.

[![Repo](https://img.shields.io/badge/→_Repository-4fc3f7?style=flat-square&labelColor=0a1628)](https://github.com/QHX-BNU/EduData)

</td>
</tr>
</table>

</td>
</tr>
</table>

<br>

<!-- ============================================ -->
<!-- CONTACT SECTION -->
<!-- ============================================ -->

<table>
<tr><td>

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ◆  CONNECT                                                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

</td></tr>
</table>

<div align="center">

```
   _________________________________________________________
  |                                                         |
  |   📧  huixingq@mail.ustc.edu.cn                         |
  |   🐙  github.com/QHX-BNU                                |
  |                                                         |
  |   LOCATION  ::  Anhui, China                            |
  |   AFFILIATION ::  University of Science and Technology   |
  |                   of China                               |
  |_________________________________________________________|
  
     ▸  OPEN TO RESEARCH COLLABORATION
```

</div>

<br>

<!-- FOOTER -->
<div align="center">

```
                    ─────────────────────────
                    ▎  SYS :: SIGNAL ACTIVE  ▎
                    ─────────────────────────
```

</div>

<!-- 
  GitHub Profile README — Holographic Projection Theme
  Huixing Que (阙辉兴) · USTC · ACL 2026
-->