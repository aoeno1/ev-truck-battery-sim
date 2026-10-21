# 无头浏览器确定性回归验证法

> 适用于一切"带动画/物理步进的 HTML 仿真页"。核心问题：**headless Edge 的 `requestAnimationFrame` 在 `--virtual-time-budget` 下完全不回调**，`--dump-dom` 只能验证初始渲染——时间相关行为全是未验证状态。

## 1. 页面必须暴露确定性步进钩子

在仿真页里固定暴露：

```js
window.__sim = { S: state, PACKS: PACKS, step: function(sec){ stepFrame(sec*60); render(); } };
```

`step(sec)` = 推进 sec 秒实时（= sec 分钟仿真）后立即重渲染。探针脚本调它，再读 DOM 断言。

## 2. 探针脚本模板（Node 驱动 Edge）

```js
const fs=require('fs'), cp=require('child_process');
const src=fs.readFileSync('<页面.html>','utf8');
const MSEDGE='C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe';
const PRE='<div id="out" style="display:none"></div><scr'+'ipt>window.addEventListener("load",function(){try{\n';
const POST='\n}catch(e){document.getElementById("out").textContent="ERR:"+e.message}});</scr'+'ipt>';
function run(tag, body){
  fs.writeFileSync('_v.html', src.replace('</body>', PRE+body+POST+'</body>'));
  cp.execSync('"'+MSEDGE+'" --headless=new --disable-gpu --no-sandbox --virtual-time-budget=6000 --dump-dom "file:///E:/<abs>/_v.html" > _vd.html',{stdio:'pipe'});
  const d=fs.readFileSync('_vd.html','utf8');
  const m=d.match(/id="out"[^>]*>([\s\S]*?)<\/div>/);
  console.log('== '+tag+'\n   '+(m?m[1].replace(/&lt;/g,'<').replace(/&amp;/g,'&'):'(no output)'));
}
// 用法：run('工况名', 'var S=__sim.S; S.pack="lfp"; S.temp=-20; S.tb=-20; S.mode="charge"; S.tmOn=false; __sim.step(60); document.getElementById("out").textContent = document.getElementById("mChgP").textContent;');
```

要点：
- 临时 HTML 用**绝对路径**传 Edge（相对路径 `--screenshot` 不落盘）
- 探针 body 里先设状态再 `step()`，最后把断言值写进 `#out`
- 一次 `run()` 一个工况，输出按 `== 标签` 分组

## 3. 四组标准工况（重卡电池仿真必跑）

| # | 工况 | 断言点 |
|---|---|---|
| 1 | 四包 25℃ 满载静态 | 电量/质量/续航/电耗/充电时长与 `battery-data-bench.md` §5 派生表逐项一致 |
| 2 | 四包 −20℃ 热管理 ON（step 240） | 容量保持率 96/97/100/99%（LFP/NMC/SS/Na），电池被拉回 17–25℃ |
| 3 | 钠电 vs LFP −20℃ 充电 热管理 OFF（step 60） | Na 600kW 满功率免预热；LFP 涓流 49kW + "低温禁充"chip |
| 4 | LFP 25℃ 600kW 热管理 OFF（step 60） | 47℃ 触发降流 30%（180kW），逼近 52℃ 停充 |

## 4. 布局回归（一屏硬约束）

```js
// 14 视口逐一验证，scrollHeight 必须 ≤ innerHeight
[1920,1600,1440,1280,1200,1199,1100,1000,900,820,780,779,700,500,420,390].forEach(w=>{...});
```

判定：`document.documentElement.scrollHeight <= window.innerHeight` 且 `scrollWidth <= innerWidth`（零滚动零横溢）。

## 5. 曾靠此法抓到的 bug（说明它不可省）

- `thermalStep` 漏除 60：热动力学快 60 倍，10 秒冲 52℃——dump-dom 完全看不出，轨迹回归立即暴露
- 充电产热错用踏板功率而非实充功率：冷态门限激活时高估产热
- 数据偏离市面：固态 280→320、钠电 131→140 Wh/kg，派生续航全变
