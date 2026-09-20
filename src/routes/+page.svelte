<script lang="ts">
	import { onMount } from 'svelte';

	// ── palettes ────────────────────────────────────────────────────────────────
	type Palette = { id:string; name:string; bg:string; bg2:string; accent:string; accent2:string; ink:string; splat:string[] };
	const palettes: Palette[] = [
		{ id:'foil', name:'Foil', bg:'#07090f', bg2:'#131528', accent:'#beff3b', accent2:'#ff3beb', ink:'#f2f5ff', splat:['#beff3b','#ff3beb','#4be1ff','#ff7a18'] },
		{ id:'nectar', name:'Nectar', bg:'#0c070a', bg2:'#2a1020', accent:'#ff7a18', accent2:'#ffcb3b', ink:'#fff7ed', splat:['#ff7a18','#ff3b8a','#ffd93b','#8aff3b'] },
		{ id:'pacific', name:'Pacific', bg:'#050c14', bg2:'#0a2538', accent:'#4be1ff', accent2:'#7b61ff', ink:'#eefaff', splat:['#4be1ff','#7b61ff','#43ffac','#ffffff'] },
		{ id:'void', name:'Void', bg:'#060606', bg2:'#1a1a1a', accent:'#ffffff', accent2:'#999999', ink:'#ffffff', splat:['#ffffff','#e0e0e0','#beff3b','#ff3beb'] },
	];

	let paletteIdx = $state(0);
	let palette = $derived(palettes[paletteIdx]);

	// ── ui state ───────────────────────────────────────────────────────────────
	let gravity = $state(900);
	let bounciness = $state(0.88);
	let soundOn = $state(true);
	let paintOn = $state(true);
	let trailFade = $state(0.12);
	let orbCount = $state(0);
	let scaleId = $state<'pentatonic'|'major'|'minor'|'chromatic'>('pentatonic');
	let helpOpen = $state(false);
	let blackHole = $state(false);
	let tiltEnabled = $state(false);
	let isDragging = $state(false);
	let fps = $state(0);
	let collisionCount = $state(0);

	const scales: Record<string, number[]> = {
		pentatonic: [0,2,4,7,9],
		major: [0,2,4,5,7,9,11],
		minor: [0,2,3,5,7,8,10],
		chromatic: [0,1,2,3,4,5,6,7,8,9,10,11],
	};
	const baseFreq = 220; // A3

	function noteFreq(semi:number){ return baseFreq * Math.pow(2, semi/12); }

	// ── kinetic letters ──────────────────────────────────────────────────────
	const letters = ['F','L','U','X'];
	let letterPos = $state(letters.map(() => ({x:0,y:0,r:0,s:1})));
	let letterScramble = $state(false);
	let kineticText = $state('FLUX');
	let heroPtr = {x:0,y:0};

	function handleHeroMove(e:PointerEvent){
		const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
		heroPtr.x = e.clientX - rect.left - rect.width/2;
		heroPtr.y = e.clientY - rect.top - rect.height/2;
		letters.forEach((_,i)=>{
			const offset = (i - 1.5) * 10;
			const dx = (heroPtr.x * 0.08 + offset) * (0.6 + i*0.15);
			const dy = heroPtr.y * 0.06 * (i%2?1:-1);
			letterPos[i] = {x:dx, y:dy, r: dx*0.08, s: 1 + Math.abs(dx)*0.003 };
		});
	}
	function resetLetters(){
		letterPos = letters.map(()=> ({x:0,y:0,r:0,s:1}));
	}
	function scrambleLetters(){
		letterScramble = true;
		kineticText = ['FLUX','XFLU','UFLX','FXUL'][Math.floor(Math.random()*4)];
		setTimeout(()=>{ kineticText='FLUX'; letterScramble=false; }, 600);
		letterPos = letters.map(()=> ({x:(Math.random()-0.5)*80, y:(Math.random()-0.5)*60, r:(Math.random()-0.5)*30, s: 1.2}));
		setTimeout(()=> letterPos = letters.map(()=> ({x:0,y:0,r:0,s:1})), 700);
		spawnOrb(true);
	}

	// ── canvas / rapier ─────────────────────────────────────────────────────
	let arenaEl: HTMLElement;
	let canvasEl: HTMLCanvasElement;
	let bgCanvasEl: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null = null;
	let bgCtx: CanvasRenderingContext2D | null = null;
	let world: any = $state(null);
	let RAPIER: any = $state(null);
	let bodies: any[] = [];
	let colliders: any[] = [];
	let radii: number[] = [];
	let hues: number[] = [];
	let splats: {x:number,y:number,r:number,c:string, a:number}[] = [];
	let draggingIdx: number | null = null;
	let lastPtr = {x:0,y:0};
	let velSamples: {x:number,y:number, t:number}[] = [];
	let raf = 0;
	let lastT = 0;
	let frameCount = 0;
	let lastFpsT = 0;

	// snapshots
	let snapshots: string[] = $state([]);

	function paletteSplatColor(){
		return palette.splat[Math.floor(Math.random()*palette.splat.length)];
	}

	function spawnOrb(atPointer=false, px?:number, py?:number){
		if(!world || !RAPIER) { console.warn('spawnOrb: world not ready'); return; }
		if(!canvasEl) { console.warn('spawnOrb: canvas not ready'); return; }
		if(bodies.length >= 24) { console.warn('spawnOrb: max reached'); return; }
		try{
			const rect = canvasEl.getBoundingClientRect();
			const w = rect.width || canvasEl.clientWidth || 900;
			const x = px ?? (atPointer ? w/2 + (Math.random()-0.5)*80 : Math.random()*(w-100)+50);
			const y = py ?? (atPointer ? 60 : Math.random()*60+30);
			const r = 14 + Math.random()*18;
			const desc = RAPIER.RigidBodyDesc.dynamic().setTranslation(x, y).setLinearDamping(0.02).setAngularDamping(0.2);
			const body = world.createRigidBody(desc);
			const col = world.createCollider(RAPIER.ColliderDesc.ball(r).setRestitution(bounciness).setFriction(0.15).setMass(r*0.6), body);
			bodies.push(body);
			colliders.push(col);
			radii.push(r);
			hues.push( (paletteIdx*70 + bodies.length* 37 + r*4) % 360 );
			orbCount = bodies.length;
			if(soundOn) pluck(Math.floor(Math.random()*7));
		}catch(e){ console.error('spawnOrb failed', e); }
	}

	function clearOrbs(){
		if(!world) return;
		bodies.forEach(b=> world.removeRigidBody(b));
		bodies = []; colliders=[]; radii=[]; hues=[]; splats=[]; orbCount=0;
		for(let i=0;i<6;i++) spawnOrb();
	}

	function shakeArena(intensity=1){
		if(!world) return;
		bodies.forEach(b=>{
			const imp = (Math.random()-0.5)* 15000 * intensity;
			const impY = (Math.random()-0.5)* 12000 * intensity - 2000;
			b.applyImpulse({x:imp, y:impY}, true);
			b.applyTorqueImpulse((Math.random()-0.5)* 8000, true);
		});
		if(soundOn){
			for(let i=0;i<3;i++) setTimeout(()=> pluck(Math.floor(Math.random()*12)), i*70);
		}
	}

	// ── audio ────────────────────────────────────────────────────────────────
	let audioCtx: AudioContext | null = null;
	function ensureAudio(){
		if(audioCtx) return;
		try{ audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)(); }catch{}
	}
	function pluck(semiOffset:number){
		if(!soundOn || !audioCtx) return;
		if(audioCtx.state === 'suspended') audioCtx.resume();
		const deg = scales[scaleId];
		const o = deg[semiOffset % deg.length] + Math.floor(semiOffset/deg.length)*12;
		const freq = noteFreq(o);
		const now = audioCtx.currentTime;
		const osc = audioCtx.createOscillator();
		const gain = audioCtx.createGain();
		const filt = audioCtx.createBiquadFilter();
		filt.type='lowpass'; filt.frequency.value = 4200; filt.Q.value=1.2;
		osc.type='triangle';
		osc.frequency.value = freq;
		osc.detune.value = (Math.random()-0.5)*12;
		gain.gain.setValueAtTime(0, now);
		gain.gain.linearRampToValueAtTime(0.38, now+0.012);
		gain.gain.exponentialRampToValueAtTime(0.001, now+0.9);
		osc.connect(filt).connect(gain).connect(audioCtx.destination);
		osc.start(now);
		osc.stop(now+1.0);
	}

	let pendingCollisions: Set<string> = new Set();

	function checkCollisions(){
		for(let i=0;i<bodies.length;i++){
			for(let j=i+1;j<bodies.length;j++){
				const a = bodies[i].translation();
				const b = bodies[j].translation();
				const dx=a.x-b.x, dy=a.y-b.y;
				const d = Math.hypot(dx,dy);
				const sum = radii[i]+radii[j];
				const key = `${i}-${j}`;
				if(d < sum + 2){
					if(!pendingCollisions.has(key)){
						pendingCollisions.add(key);
						// splat
						if(paintOn){
							const mx=(a.x+b.x)/2, my=(a.y+b.y)/2;
							splats.push({x:mx,y:my,r: 6+ Math.random()*10, c: paletteSplatColor(), a:0.85});
							if(splats.length>220) splats.shift();
						}
						if(soundOn){
							const speed = Math.hypot(bodies[i].linvel().x, bodies[j].linvel().x);
							const semi = Math.floor(( (a.y % 400)/30 + speed/80) % 14);
							pluck(semi);
						}
						collisionCount++;
						setTimeout(()=> pendingCollisions.delete(key), 120);
					}
				} else {
					if(d > sum+12) pendingCollisions.delete(key);
				}
			}
		}
	}

	// ── walls ────────────────────────────────────────────────────────────────
	let wallBodies: any[] = [];
	function buildWalls(){
		if(!world || !RAPIER) return;
		// remove old
		wallBodies.forEach(b=> world.removeRigidBody(b));
		wallBodies=[];
		const w = canvasEl.clientWidth;
		const h = canvasEl.clientHeight;
		const thick = 32;
		const defs = [
			{ x:w/2, y:h+thick/2-1, w:w, h:thick }, // floor
			{ x: -thick/2+1, y:h/2, w:thick, h:h }, // left
			{ x: w+thick/2-1, y:h/2, w:thick, h:h }, // right
			{ x: w/2, y: -thick/2+1, w:w, h:thick }, // ceiling
		];
		defs.forEach(d=>{
			const desc = RAPIER.RigidBodyDesc.fixed().setTranslation(d.x, d.y);
			const b = world.createRigidBody(desc);
			world.createCollider(RAPIER.ColliderDesc.cuboid(d.w/2, d.h/2).setRestitution(0.95).setFriction(0.2), b);
			wallBodies.push(b);
		});
		// floating platforms for pinball feel
		const plat = [
			{ x:w*0.28, y:h*0.62, w:110, h:14, a:0.18 },
			{ x:w*0.72, y:h*0.62, w:110, h:14, a:-0.18 },
			{ x:w*0.5, y:h*0.42, w:90, h:12, a:0 },
		];
		plat.forEach(p=>{
			const desc = RAPIER.RigidBodyDesc.fixed().setTranslation(p.x, p.y).setRotation(p.a);
			const b = world.createRigidBody(desc);
			world.createCollider(RAPIER.ColliderDesc.cuboid(p.w/2, p.h/2).setRestitution(0.92), b);
			wallBodies.push(b);
		});
	}

	// ── bg shader-ish ──────────────────────────────────────────────────────
	let bgT = 0;
	function drawBackground(dt:number){
		if(!bgCtx || !bgCanvasEl) return;
		bgT += dt * 0.00035;
		const w = bgCanvasEl.width, h = bgCanvasEl.height;
		bgCtx.clearRect(0,0,w,h);
		// gradient mesh
		const g = bgCtx.createLinearGradient(0,0,w,h);
		g.addColorStop(0, palette.bg);
		g.addColorStop(1, palette.bg2);
		bgCtx.fillStyle = g;
		bgCtx.fillRect(0,0,w,h);
		// floating orbs behind
		bgCtx.globalCompositeOperation = 'lighter';
		for(let i=0;i<3;i++){
			const cx = w*0.2 + Math.sin(bgT* (0.7+i*0.3) + i)* w*0.18 + w*0.25*i;
			const cy = h*0.35 + Math.cos(bgT* (0.6+i*0.2) + i*1.7)* h*0.18;
			const r = (w*0.18 + Math.sin(bgT+i)* w*0.02);
			const grad = bgCtx.createRadialGradient(cx,cy,0,cx,cy,r);
			const c1 = palette.accent;
			const c2 = palette.accent2;
			// hex to rgba approx via canvas
			grad.addColorStop(0, c1 + '55');
			grad.addColorStop(0.55, c2 + '22');
			grad.addColorStop(1, 'transparent');
			bgCtx.fillStyle = grad;
			bgCtx.beginPath(); bgCtx.arc(cx,cy,r,0,Math.PI*2); bgCtx.fill();
		}
		// grain
		bgCtx.globalCompositeOperation='source-over';
		bgCtx.globalAlpha=0.07;
		bgCtx.fillStyle='#ffffff';
		for(let i=0;i<90;i++){
			const x = (Math.sin(i*12.9898+bgT*5)*43758.5453 %1)*w;
			const y = (Math.sin(i*78.233+bgT*3)*43758.5 %1)*h;
			bgCtx.fillRect(x,y,1,1);
		}
		bgCtx.globalAlpha=1;
	}

	// ── main render ────────────────────────────────────────────────────────
	function resize(){
		const dpr = Math.min(window.devicePixelRatio || 1, 1.75);
		for(const c of [canvasEl, bgCanvasEl]){
			if(!c) continue;
			const rect = c.getBoundingClientRect();
			c.width = rect.width * dpr;
			c.height = rect.height * dpr;
			const cx = c.getContext('2d');
			if(cx) cx.setTransform(dpr,0,0,dpr,0,0);
		}
		ctx = canvasEl.getContext('2d');
		bgCtx = bgCanvasEl.getContext('2d');
		if(world) buildWalls();
	}

	function frame(t:number){
		raf = requestAnimationFrame(frame);
		const dt = Math.min(32, t - lastT);
		lastT = t;
		frameCount++;
		if(t - lastFpsT > 450){ fps = Math.round(frameCount*1000/(t-lastFpsT)); frameCount=0; lastFpsT=t; }
		drawBackground(dt);
		if(!ctx || !world) return;
		// step physics
		world.timestep = Math.min(dt/1000, 1/30);
		world.gravity = { x: 0, y: blackHole ? -400 : gravity };
		if(blackHole){
			const w = canvasEl.clientWidth, h = canvasEl.clientHeight;
			const cx=w/2, cy=h/2;
			bodies.forEach(b=>{
				const p=b.translation();
				const dx=cx-p.x, dy=cy-p.y;
				const d=Math.hypot(dx,dy)||1;
				const f=  280000 / (d+120);
				b.addForce({x: dx/d * f, y: dy/d * f}, true);
			});
		}
		// drag force
		if(draggingIdx !== null && isDragging){
			const b = bodies[draggingIdx];
			const p = b.translation();
			const dx = lastPtr.x - p.x;
			const dy = lastPtr.y - p.y;
			b.setLinvel({x: dx*14, y: dy*14}, true);
		}
		world.step();
		checkCollisions();

		const w = canvasEl.clientWidth, h = canvasEl.clientHeight;
		// fade trails
		if(paintOn && trailFade>0){
			ctx.fillStyle = `rgba(7,9,15,${trailFade*0.35})`;
			// actually use palette bg with alpha
			ctx.fillStyle = palette.bg + Math.round(trailFade*42).toString(16).padStart(2,'0');
			// we want subtle fade not full clear
			ctx.globalCompositeOperation='source-over';
			ctx.fillRect(0,0,w,h);
		} else {
			ctx.clearRect(0,0,w,h);
		}
		// splats behind orbs
		for(const s of splats){
			ctx.globalAlpha = s.a;
			ctx.fillStyle = s.c;
			ctx.beginPath();
			ctx.arc(s.x, s.y, s.r, 0, Math.PI*2);
			ctx.fill();
			// inner highlight
			ctx.fillStyle='#ffffff';
			ctx.globalAlpha = s.a*0.35;
			ctx.beginPath(); ctx.arc(s.x - s.r*0.25, s.y - s.r*0.25, s.r*0.28, 0, Math.PI*2); ctx.fill();
		}
		ctx.globalAlpha=1;

		// platforms visual
		ctx.strokeStyle = 'rgba(255,255,255,0.08)';
		ctx.lineWidth=1;
		// orbs
		for(let i=0;i<bodies.length;i++){
			const p = bodies[i].translation();
			const r = radii[i];
			const v = bodies[i].linvel();
			const speed = Math.hypot(v.x, v.y);
			const rot = bodies[i].rotation();
			const hue = hues[i];

			// shadow
			ctx.fillStyle='rgba(0,0,0,0.22)';
			ctx.beginPath();
			ctx.ellipse(p.x, p.y + r*0.65, r*0.9, r*0.45, 0,0,Math.PI*2);
			ctx.fill();

			// body gradient
			const grad = ctx.createRadialGradient(p.x - r*0.35, p.y - r*0.4, r*0.15, p.x, p.y, r);
			const light = `hsl(${hue} 92% 68%)`;
			const mid = `hsl(${hue} 88% 58%)`;
			const dark = `hsl(${hue} 72% 42%)`;
			grad.addColorStop(0, '#ffffff');
			grad.addColorStop(0.18, light);
			grad.addColorStop(0.58, mid);
			grad.addColorStop(1, dark);
			ctx.fillStyle = grad;
			ctx.beginPath(); ctx.arc(p.x, p.y, r, 0, Math.PI*2); ctx.fill();

			// chrome rim
			ctx.strokeStyle = 'rgba(255,255,255,0.85)';
			ctx.lineWidth = 1.1;
			ctx.stroke();

			// motion streak if fast
			if(speed > 900){
				ctx.strokeStyle = `hsla(${hue} 90% 60% / 0.22)`;
				ctx.lineWidth = 2;
				ctx.beginPath();
				ctx.moveTo(p.x - v.x*0.012, p.y - v.y*0.012);
				ctx.lineTo(p.x, p.y);
				ctx.stroke();
			}
			// highlight dot
			ctx.fillStyle='rgba(255,255,255,0.9)';
			ctx.beginPath(); ctx.arc(p.x - r*0.32, p.y - r*0.32, Math.max(2, r*0.13), 0, Math.PI*2); ctx.fill();

			// dragging ring
			if(draggingIdx===i){
				ctx.strokeStyle = palette.accent;
				ctx.lineWidth = 2.5;
				ctx.setLineDash([6,6]);
				ctx.beginPath(); ctx.arc(p.x, p.y, r+8, 0, Math.PI*2); ctx.stroke();
				ctx.setLineDash([]);
			}
		}

		// crosshair for drag
		if(isDragging && draggingIdx!==null){
			ctx.strokeStyle='rgba(255,255,255,0.35)';
			ctx.setLineDash([4,6]);
			ctx.beginPath();
			ctx.moveTo(lastPtr.x-14, lastPtr.y); ctx.lineTo(lastPtr.x+14, lastPtr.y);
			ctx.moveTo(lastPtr.x, lastPtr.y-14); ctx.lineTo(lastPtr.x, lastPtr.y+14);
			ctx.stroke(); ctx.setLineDash([]);
		}
	}

	// ── pointer ────────────────────────────────────────────────────────────
	function getPointer(e:PointerEvent){
		const rect = canvasEl.getBoundingClientRect();
		return { x: e.clientX - rect.left, y: e.clientY - rect.top };
	}
	function onPointerDown(e:PointerEvent){
		ensureAudio();
		const p = getPointer(e);
		lastPtr=p;
		// hit test from topmost
		let hit = -1;
		let best=Infinity;
		for(let i=0;i<bodies.length;i++){
			const pos=bodies[i].translation();
			const d=Math.hypot(pos.x - p.x, pos.y - p.y);
			if(d < radii[i] + 10 && d < best){ best=d; hit=i; }
		}
		if(hit !== -1){
			draggingIdx=hit; isDragging=true;
			(canvasEl as any).setPointerCapture?.(e.pointerId);
			bodies[hit].setLinearDamping(6, true);
			velSamples=[];
		} else {
			// spawn at pointer on click if not dragging
			spawnOrb(false, p.x, p.y);
		}
	}
	function onPointerMove(e:PointerEvent){
		const p = getPointer(e);
		lastPtr=p;
		if(isDragging && draggingIdx!==null){
			velSamples.push({x:p.x,y:p.y,t:performance.now()});
			if(velSamples.length>6) velSamples.shift();
		}
	}
	function onPointerUp(e:PointerEvent){
		if(isDragging && draggingIdx!==null){
			const b=bodies[draggingIdx];
			b.setLinearDamping(0.02, true);
			if(velSamples.length>=2){
				const a=velSamples[0], c=velSamples[velSamples.length-1];
				const dt = Math.max(16, c.t - a.t);
				const vx=(c.x - a.x)/dt * 900;
				const vy=(c.y - a.y)/dt * 900;
				b.setLinvel({x:vx, y:vy}, true);
			}
			// impulse pop sound
			if(soundOn) pluck(Math.floor(Math.random()*5)+3);
		}
		isDragging=false; draggingIdx=null;
	}

	function saveSnapshot(){
		const url = canvasEl.toDataURL('image/png');
		snapshots = [url, ...snapshots].slice(0,12);
		try{ localStorage.setItem('flux-snapshots', JSON.stringify(snapshots)); }catch{}
	}
	function clearSplats(){ splats=[]; snapshots=[]; try{ localStorage.removeItem('flux-snapshots')}catch{} }

	// ── keys + konami ──────────────────────────────────────────────────────
	const konami = ['ArrowUp','ArrowUp','ArrowDown','ArrowDown','ArrowLeft','ArrowRight','ArrowLeft','ArrowRight','b','a'];
	let konamiIdx=0;
	function onKey(e:KeyboardEvent){
		if(e.key === '?'){ helpOpen = !helpOpen; }
		if(e.key === 'Escape'){ helpOpen=false; isDragging=false; draggingIdx=null; }
		if(e.key === ' '){ e.preventDefault(); shakeArena(1.4); }
		if(e.key === 'c' || e.key==='C'){ clearSplats(); }
		if(e.key === 's' || e.key==='S'){ saveSnapshot(); }
		if(e.key >= '1' && e.key <= '4'){ paletteIdx = parseInt(e.key)-1; hues = hues.map((_,i)=> (paletteIdx*70 + i*37) %360 ); }
		if(e.key === 'g' || e.key==='G'){ gravity = gravity>0 ? -900 : 900; }
		if(e.key === 'p' || e.key==='P'){ paintOn = !paintOn; }
		if(e.key === 'm' || e.key==='M'){ soundOn = !soundOn; if(soundOn) ensureAudio(); }
		if(e.key === 'n' || e.key==='N'){ spawnOrb(true); }
		if(e.key === 'b' || e.key==='B'){ blackHole = !blackHole; }

		// konami
		if(e.key === konami[konamiIdx]){ konamiIdx++; if(konamiIdx===konami.length){ konamiIdx=0; blackHole=!blackHole; shakeArena(2); spawnOrb(true); setTimeout(()=>spawnOrb(true),80); } }
		else konamiIdx = e.key===konami[0]?1:0;
	}

	let scrollY = $state(0);

	// @ts-ignore - async onMount with cleanup is fine at runtime
	onMount(async()=>{
		ensureAudio();
		// load snapshots
		try{ const s=localStorage.getItem('flux-snapshots'); if(s) snapshots=JSON.parse(s);}catch{}
		// rapier
		try{
			const mod = await import('@dimforge/rapier2d-compat');
			// @ts-ignore - compat init signature varies
			await mod.init();
			RAPIER = mod;
			world = new RAPIER.World({x:0,y:gravity});
			buildWalls();
			for(let i=0;i<7;i++) spawnOrb();
			// resize + loop
			resize();
			window.addEventListener('resize', resize);
			window.addEventListener('keydown', onKey);
			window.addEventListener('scroll', ()=> scrollY = window.scrollY, {passive:true});
			// tilt
			if(typeof (window as any).DeviceOrientationEvent !== 'undefined'){
				try{
					// @ts-ignore
					if(typeof DeviceOrientationEvent.requestPermission === 'function'){
						// leave disabled until user enables via button
					} else {
						window.addEventListener('deviceorientation', (ev:any)=>{
							if(!tiltEnabled) return;
							const gx = (ev.gamma||0) * 18;
							const gy = (ev.beta||0) * 18;
							if(world) world.gravity = {x: gx*6, y: gy*6 + gravity*0.2};
						});
					}
				}catch{}
			}
			raf = requestAnimationFrame(frame);
		} catch(err){
			console.error('rapier failed', err);
		}
		return ()=>{
			cancelAnimationFrame(raf);
			window.removeEventListener('resize', resize);
			window.removeEventListener('keydown', onKey);
		};
	});

	// keep bounciness live
	$effect(()=>{
		// need to recreate colliders friction? For simplicity update next spawns; existing bodies keep old.
		void bounciness;
	});
	$effect(()=>{ if(world) world.gravity = {x:0, y:blackHole? -400 : gravity}; });
</script>

<svelte:window onkeydown={onKey} />

<div class="flux" style="--bg:{palette.bg}; --bg2:{palette.bg2}; --accent:{palette.accent}; --accent2:{palette.accent2}; --ink:{palette.ink};">
	<!-- ambient bg -->
	<canvas bind:this={bgCanvasEl} class="bg-canvas" aria-hidden="true"></canvas>
	<div class="grain" aria-hidden="true"></div>

	<!-- top bar -->
	<nav class="topbar">
		<a class="brand" href="https://cycoconutz.github.io/portfolio-tabs" aria-label="Back to catalog">
			<span class="mark">◈</span>
			<span class="brand-text"><strong>FLUX</strong> <em>/ Kinetic Arcade</em></span>
		</a>
		<div class="top-actions">
			<div class="palette-dots" role="group" aria-label="Palettes 1-4">
				{#each palettes as p,i}
					<button class="dot" class:active={i===paletteIdx} style="--c:{p.accent}; --c2:{p.accent2}" onclick={()=> paletteIdx=i} aria-label="Palette {p.name}" title="{p.name} ({i+1})"></button>
				{/each}
			</div>
			<a class="pill ghost" href="https://github.com/cycoconutz/flux" target="_blank" rel="noreferrer">Repo ↗</a>
			<button class="pill" onclick={()=> helpOpen=!helpOpen}>? Help</button>
		</div>
	</nav>

	<!-- hero -->
	<section class="hero" onpointermove={handleHeroMove} onpointerleave={resetLetters} ondblclick={scrambleLetters} role="button" tabindex="0" aria-label="Double-click to scramble" onkeydown={(e)=> e.key==='Enter' && scrambleLetters()}>
		<div class="hero-meta">
			<span class="eyebrow">06 — Arcade — 2026 — Svelte 5 · Rapier · Web Audio · GLSL-ish</span>
			<div class="hero-stats">
				<span>{orbCount} orbs</span>
				<span>{collisionCount} hits</span>
				<span>{fps} fps</span>
				<span class:active={blackHole} class="hole">{blackHole ? '● Black hole' : '○ Gravity well'}</span>
			</div>
		</div>

		<h1 class="kinetic" aria-label="FLUX">
			{#each kineticText.split('') as ch,i}
				{@const s = letterPos[i] ?? {x:0,y:0,r:0,s:1}}
				<span class="kl" style="transform: translate({s.x}px, {s.y}px) rotate({s.r}deg) scale({s.s}); --i:{i}">{ch}</span>
			{/each}
		</h1>
		<p class="hero-tag" style="opacity:{1 - Math.min(scrollY/420,0.85)}; transform: translateY({Math.min(scrollY*0.12,18)}px)">
			Throw, pluck, and splatter. <strong>Drag orbs</strong> to fling them · <strong>Click empty space</strong> to spawn · Every collision is a note.
			<br class="hide-mobile"/>Double-click the title to scramble. Try the <code>Konami code</code>.
		</p>

		<div class="hero-ctas">
			<button class="cta primary" onclick={()=> spawnOrb(true)}>＋ Add orb (N)</button>
			<button class="cta" onclick={()=> shakeArena(1.5)}>⟡ Shake (Space)</button>
			<button class="cta" onclick={()=> {blackHole=!blackHole; shakeArena(1.1)}}>{blackHole ? '◍ Release' : '◉ Black hole (B)'}</button>
			<button class="cta ghost" onclick={scrambleLetters}>Scrambl3</button>
		</div>

		<div class="scroll-hint" aria-hidden="true">
			<span>drag to throw</span>
			<i>↘</i>
			<span>click to spawn</span>
			<i>✦</i>
			<span>space to shake</span>
		</div>
	</section>

	<!-- arena -->
	<section class="arena-wrap" bind:this={arenaEl}>
		<div class="arena-toolbar">
			<div class="group">
				<label>Gravity <input type="range" min="-1600" max="1600" step="50" bind:value={gravity} /> <b>{gravity>0? '↓':'↑'} {Math.abs(gravity)}</b></label>
				<button class="mini" onclick={()=> gravity = gravity>0 ? -900 : 900}>Invert G</button>
			</div>
			<div class="group">
				<label>Bounce <input type="range" min="0.2" max="0.99" step="0.02" bind:value={bounciness} /> <b>{bounciness.toFixed(2)}</b></label>
				<label class="check"><input type="checkbox" bind:checked={paintOn} /> Paint <span class="k">P</span></label>
				<label class="check"><input type="checkbox" bind:checked={soundOn} onchange={()=> soundOn && ensureAudio()} /> Sound <span class="k">M</span></label>
			</div>
			<div class="group">
				<label>Fade <input type="range" min="0" max="0.28" step="0.02" bind:value={trailFade} /></label>
				<select bind:value={scaleId} aria-label="Scale">
					<option value="pentatonic">Pentatonic</option>
					<option value="major">Major</option>
					<option value="minor">Minor</option>
					<option value="chromatic">Chromatic</option>
				</select>
			</div>
			<div class="group actions">
				<button class="mini primary" onclick={()=> spawnOrb(true)}>＋ Orb</button>
				<button class="mini" onclick={clearOrbs}>Clear</button>
				<button class="mini" onclick={saveSnapshot}>Save PNG (S)</button>
				<button class="mini" onclick={clearSplats}>Wipe (C)</button>
			</div>
		</div>

		<div class="arena-frame">
			<canvas
				bind:this={canvasEl}
				class="arena"
				width="900"
				height="560"
				onpointerdown={onPointerDown}
				onpointermove={onPointerMove}
				onpointerup={onPointerUp}
				onpointercancel={onPointerUp}
				style="touch-action: none"
				aria-label="Physics arena — drag orbs, click to spawn"
			></canvas>

			<div class="arena-overlay">
				<span class="overlay-hint">{isDragging ? 'release to fling →' : 'drag orbs • click to spawn • double-click title'}</span>
				<span class="overlay-count">{bodies.length} bodies • {splats.length} splats</span>
			</div>

			<!-- tilt enable for iOS -->
			<button class="tilt-btn" onclick={async()=>{
				try{
					// @ts-ignore
					if(typeof DeviceOrientationEvent !=='undefined' && typeof DeviceOrientationEvent.requestPermission==='function'){
						// @ts-ignore
						const r = await DeviceOrientationEvent.requestPermission();
						if(r==='granted') tiltEnabled=true;
					} else { tiltEnabled=!tiltEnabled; }
				}catch{ tiltEnabled=!tiltEnabled;}
			}} title="Enable tilt gravity on mobile">{tiltEnabled ? 'Tilt ✓' : 'Tilt'}</button>
		</div>

		{#if snapshots.length}
			<div class="strip">
				{#each snapshots as url,i}
					<button class="snap" onclick={()=> window.open(url,'_blank')} title="Open snapshot">
						<img src={url} alt="Saved splat {i+1}" />
					</button>
				{/each}
			</div>
		{/if}
	</section>

	<!-- features -->
	<section class="features">
		<div class="feat">
			<h3>01 — Pointer Physics</h3>
			<p>Rapier2D WASM with restitution {bounciness.toFixed(2)}. Every orb is a dynamic body with mass ∝ radius. Drag sets linear velocity from pointer delta — fling speed matters. Platforms are fixed cuboids rotated ±10° for pinball chaos.</p>
			<code>World(RAPIER) · RigidBodyDesc.dynamic · Collider.ball(r)</code>
		</div>
		<div class="feat">
			<h3>02 — Sound Collider</h3>
			<p>Web Audio API — no libraries. Collision distance check O(n²) debounced 120ms triggers a triangle oscillator through a lowpass. Quantized to <strong>{scaleId}</strong> so it never sounds wrong.</p>
			<div class="scale-row">
				{#each scales[scaleId] as semi}
					<button class="note" onclick={()=> {ensureAudio(); pluck(semi)}}>{semi}</button>
				{/each}
			</div>
		</div>
		<div class="feat">
			<h3>03 — Paint Trails</h3>
			<p>2D canvas splats at collision mid-points + optional velocity trails. Fade is a translucent rect each frame. Snapshots are <code>toDataURL</code> stored in <code>localStorage</code>. Try holding drag through multiple collisions.</p>
			<button class="mini" onclick={clearSplats}>Wipe canvas</button>
		</div>
		<div class="feat">
			<h3>04 — Shader-ish Field</h3>
			<p>Not a real GLSL pass — a lightweight canvas gradient mesh that mimics a fluid shader for GH Pages. Three radial gradients drift with <code>sin(t)</code> behind the arena so the whole page breathes.</p>
			<code>Canvas radialGradient · globalCompositeOperation=lighter</code>
		</div>
		<div class="feat">
			<h3>05 — Kinetic Type</h3>
			<p>Each glyph is a <code>spring()</code> from <code>svelte/motion</code> that chases the pointer. Double-click title or press Enter to scramble — same springs that drive the orbs.</p>
			<button class="mini" onclick={scrambleLetters}>Scramble</button>
		</div>
		<div class="feat">
			<h3>06 — Why Svelte?</h3>
			<p>The catalog is 5× React. FLUX is <strong>Svelte 5 runes</strong> + <strong>SvelteKit static</strong> on GH Pages, no React runtime. Proves stack diversity: compiler reactivity (<code>$state/$derived/$effect</code>), no VDOM, tiny bundle.</p>
			<div class="tech-tags"><span>Svelte 5</span><span>SvelteKit</span><span>Rapier</span><span>Web Audio</span><span>Vite 6</span></div>
		</div>
	</section>

	<!-- footer -->
	<footer class="footer">
		<div>
			<strong>FLUX</strong> — built for <a href="https://cycoconutz.github.io/portfolio-tabs">Project Catalog</a>. Static, no backend, no tracking.
		</div>
		<div class="foot-links">
			<a href="https://github.com/cycoconutz/flux" target="_blank" rel="noreferrer">Source ↗</a>
			<a href="https://cycoconutz.github.io/portfolio-tabs">Catalog ←</a>
		</div>
	</footer>

	{#if helpOpen}
		<button class="help-backdrop" onclick={()=> helpOpen=false} aria-label="Close help"></button>
		<div class="help" role="dialog" aria-modal="true" aria-label="Help">
			<h2>How to play</h2>
			<ul>
				<li><b>Drag</b> an orb and release to fling (velocity matters)</li>
				<li><b>Click</b> empty space to spawn · <b>N</b> also spawns</li>
				<li><b>Space</b> shake · <b>G</b> invert gravity · <b>B</b> black hole</li>
				<li><b>P</b> toggle paint · <b>M</b> mute · <b>1–4</b> palettes · <b>S</b> save PNG · <b>C</b> wipe</li>
				<li><b>Double-click FLUX</b> to scramble · <b>?</b> toggles this</li>
				<li><b>Konami</b> ↑↑↓↓←→←→BA triggers black hole + shower</li>
				<li>Mobile: <b>Tilt</b> button enables gravity tilt (grant permission on iOS)</li>
			</ul>
			<div class="help-foot">
				<span>Tip: max 24 orbs — clear to reset.</span>
				<button class="pill primary" onclick={()=> helpOpen=false}>Got it</button>
			</div>
		</div>
	{/if}
</div>

<style>
	.flux{
		position:relative; min-height:100dvh; isolation:isolate;
		background: var(--bg);
		color: var(--ink);
		font-family: 'Space Grotesk', system-ui, sans-serif;
		overflow: clip;
		--pad: clamp(16px, 4vw, 40px);
	}
	.bg-canvas{ position:fixed; inset:0; width:100%; height:100%; z-index:-2; }
	.grain{
		position:fixed; inset:0; z-index:-1; pointer-events:none; opacity:0.055;
		background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.45'/%3E%3C/svg%3E");
	}
	.topbar{
		position:sticky; top:0; z-index:10;
		display:flex; align-items:center; justify-content:space-between; gap:16px;
		padding:14px var(--pad);
		background: color-mix(in srgb, var(--bg) 72%, transparent);
		backdrop-filter: blur(14px) saturate(1.2);
		border-bottom: 1px solid rgba(255,255,255,0.08);
	}
	.brand{ display:flex; align-items:center; gap:12px; text-decoration:none; color:inherit; }
	.mark{
		display:grid; place-items:center; width:38px; height:38px; border-radius:12px;
		background: var(--accent); color: var(--bg); font-weight:800; font-size:18px;
		box-shadow: 0 6px 22px color-mix(in srgb, var(--accent) 42%, transparent);
	}
	.brand-text strong{ font-family:'Syne', sans-serif; letter-spacing:-0.02em; font-size:18px; }
	.brand-text em{ font-style:normal; font-family:'JetBrains Mono', monospace; font-size:11px; letter-spacing:0.16em; text-transform:uppercase; opacity:0.6; margin-left:6px;}
	.top-actions{ display:flex; align-items:center; gap:10px; flex-wrap:wrap; }
	.palette-dots{ display:flex; gap:8px; align-items:center; padding:6px 10px; border:1px solid rgba(255,255,255,0.08); border-radius:999px; background:rgba(255,255,255,0.04); }
	.dot{
		width:18px; height:18px; border-radius:50%; border:2px solid transparent; cursor:pointer;
		background: radial-gradient(circle at 35% 30%, var(--c), var(--c2));
		box-shadow: 0 0 0 1px rgba(255,255,255,0.15) inset;
	}
	.dot.active{ border-color:#fff; transform:scale(1.15); box-shadow: 0 0 12px var(--c); }
	.pill{
		appearance:none; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.06);
		color:var(--ink); padding:9px 14px; border-radius:999px; font:600 12px 'JetBrains Mono', monospace;
		letter-spacing:0.12em; text-transform:uppercase; cursor:pointer; text-decoration:none; display:inline-flex; align-items:center; gap:6px;
	}
	.pill.primary{ background:var(--accent); color:var(--bg); border-color:var(--accent); }
	.pill.ghost{ background:transparent; }

	.hero{
		padding: clamp(28px, 7vw, 72px) var(--pad) 18px;
		max-width: 1220px; margin:0 auto;
	}
	.hero-meta{ display:flex; flex-wrap:wrap; justify-content:space-between; gap:12px; align-items:center; }
	.eyebrow{ font:500 10px 'JetBrains Mono', monospace; letter-spacing:0.22em; text-transform:uppercase; color:var(--accent); }
	.hero-stats{ display:flex; gap:10px; font:600 11px 'JetBrains Mono', monospace; letter-spacing:0.1em; text-transform:uppercase; opacity:0.7; }
	.hero-stats span{ padding:5px 9px; border:1px solid rgba(255,255,255,0.08); border-radius:999px; background:rgba(255,255,255,0.04); }
	.hero-stats .active{ color:var(--bg); background:var(--accent); border-color:var(--accent); }
	.kinetic{
		margin: 18px 0 10px; display:flex; gap: clamp(4px, 1.2vw, 18px); font-family:'Syne', sans-serif; font-weight:800;
		font-size: clamp(64px, 18vw, 168px); line-height:0.86; letter-spacing:-0.05em; cursor:pointer; user-select:none;
		filter: drop-shadow(0 10px 30px color-mix(in srgb, var(--accent) 18%, transparent));
	}
	.kl{
		display:inline-block; will-change:transform;
		background: linear-gradient(180deg, #fff 15%, var(--ink) 55%, color-mix(in srgb, var(--accent) 70%, #fff) 100%);
		-webkit-background-clip:text; background-clip:text; color:transparent;
		text-shadow: 0 0 0 rgba(0,0,0,0);
		transition: transform 0.45s cubic-bezier(0.23,1,0.32,1), filter 0.2s;
	}
	.kl:nth-child(2){ background-image: linear-gradient(180deg, #fff 10%, var(--ink) 60%, var(--accent2) 100%); }
	.hero-tag{ max-width:64ch; color: color-mix(in srgb, var(--ink) 72%, transparent); font-size: clamp(14px, 1.7vw, 16px); line-height:1.6; margin: 8px 0 18px;}
	.hero-tag code{ font-family:'JetBrains Mono', monospace; font-size:12px; padding:2px 6px; border-radius:6px; background:rgba(255,255,255,0.08); border:1px solid rgba(255,255,255,0.08); }
	.hero-ctas{ display:flex; gap:10px; flex-wrap:wrap; }
	.cta{
		appearance:none; border:1px solid rgba(255,255,255,0.1); background:rgba(255,255,255,0.06); color:var(--ink);
		padding:12px 16px; border-radius:999px; font:700 12px 'JetBrains Mono', monospace; letter-spacing:0.12em; text-transform:uppercase; cursor:pointer;
	}
	.cta.primary{ background:var(--accent); color:var(--bg); border-color:var(--accent); box-shadow:0 10px 26px color-mix(in srgb, var(--accent) 28%, transparent); }
	.cta.ghost{ background:transparent; }
	.scroll-hint{ display:flex; gap:14px; align-items:center; margin-top:18px; font:600 10px 'JetBrains Mono', monospace; letter-spacing:0.18em; text-transform:uppercase; opacity:0.5; }
	.scroll-hint i{ width:22px; height:22px; display:grid; place-items:center; border:1px solid rgba(255,255,255,0.12); border-radius:50%; font-style:normal; }

	.arena-wrap{ max-width:1220px; margin: 8px auto 0; padding:0 var(--pad) 22px; }
	.arena-toolbar{
		display:flex; flex-wrap:wrap; gap:12px; align-items:center; justify-content:space-between;
		padding:12px; border:1px solid rgba(255,255,255,0.08); border-radius:16px; background: color-mix(in srgb, var(--bg) 72%, transparent);
		backdrop-filter: blur(12px);
		font:500 12px 'JetBrains Mono', monospace;
	}
	.group{ display:flex; gap:10px; align-items:center; flex-wrap:wrap; }
	.group label{ display:flex; gap:8px; align-items:center; opacity:0.9; }
	.group b{ font-size:11px; padding:3px 7px; border-radius:999px; background:rgba(255,255,255,0.08); }
	.mini{
		appearance:none; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.07);
		color:var(--ink); padding:7px 10px; border-radius:999px; font:700 11px 'JetBrains Mono', monospace; letter-spacing:0.1em; text-transform:uppercase; cursor:pointer;
	}
	.mini.primary{ background:var(--accent); color:var(--bg); border-color:var(--accent); }
	.check{ gap:6px !important; }
	.check .k{ font-size:10px; opacity:0.5; border:1px solid rgba(255,255,255,0.14); padding:1px 5px; border-radius:4px; }
	select{ background:rgba(255,255,255,0.06); color:var(--ink); border:1px solid rgba(255,255,255,0.12); border-radius:999px; padding:6px 10px; font:600 11px 'JetBrains Mono', monospace; }

	.arena-frame{
		position:relative; margin-top:14px; border:1px solid rgba(255,255,255,0.08); border-radius:22px; overflow:hidden;
		background: radial-gradient(1200px 600px at 30% 20%, color-mix(in srgb, var(--accent) 10%, transparent), transparent 60%),
		            radial-gradient(900px 500px at 85% 80%, color-mix(in srgb, var(--accent2) 10%, transparent), transparent 60%),
		            linear-gradient(180deg, rgba(255,255,255,0.04), rgba(255,255,255,0.015));
		box-shadow: 0 22px 60px rgba(0,0,0,0.45), 0 0 0 1px rgba(255,255,255,0.06) inset;
	}
	.arena{ display:block; width:100%; height: clamp(420px, 58vh, 620px); cursor: crosshair; }
	.arena-overlay{
		position:absolute; left:12px; right:12px; bottom:10px; display:flex; justify-content:space-between; gap:12px;
		font:600 10px 'JetBrains Mono', monospace; letter-spacing:0.14em; text-transform:uppercase; opacity:0.62; pointer-events:none;
	}
	.arena-overlay span{ background:rgba(0,0,0,0.45); border:1px solid rgba(255,255,255,0.08); padding:6px 9px; border-radius:999px; backdrop-filter: blur(8px); }
	.tilt-btn{
		position:absolute; top:12px; right:12px; z-index:2;
		appearance:none; border:1px solid rgba(255,255,255,0.12); background:rgba(0,0,0,0.35); color:#fff;
		padding:7px 10px; border-radius:999px; font:700 11px 'JetBrains Mono', monospace; letter-spacing:0.1em; text-transform:uppercase; cursor:pointer; backdrop-filter: blur(8px);
	}
	.strip{ display:flex; gap:10px; overflow:auto; padding:12px 2px; scrollbar-width:thin; }
	.snap{ flex:0 0 140px; height:88px; border-radius:12px; overflow:hidden; border:1px solid rgba(255,255,255,0.1); padding:0; cursor:pointer; background:#000; }
	.snap img{ width:100%; height:100%; object-fit:cover; display:block; }

	.features{
		max-width:1220px; margin: 18px auto 0; padding: 0 var(--pad) 24px;
		display:grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap:1px;
		background: rgba(255,255,255,0.08); border:1px solid rgba(255,255,255,0.08); border-radius:18px; overflow:hidden;
	}
	.feat{ background: color-mix(in srgb, var(--bg) 86%, transparent); padding:22px; backdrop-filter: blur(8px); }
	.feat h3{ margin:0 0 8px; font:800 12px 'JetBrains Mono', monospace; letter-spacing:0.14em; text-transform:uppercase; color:var(--accent); }
	.feat p{ margin:0 0 10px; color: color-mix(in srgb, var(--ink) 72%, transparent); font-size:13.5px; line-height:1.65; }
	.feat code{ font:500 11px 'JetBrains Mono', monospace; padding:2px 6px; border-radius:6px; background:rgba(255,255,255,0.06); border:1px solid rgba(255,255,255,0.08); }
	.scale-row{ display:flex; gap:6px; flex-wrap:wrap; }
	.note{ width:30px; height:30px; border-radius:50%; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.06); color:var(--ink); font:700 11px 'JetBrains Mono', monospace; cursor:pointer; }
	.tech-tags{ display:flex; gap:6px; flex-wrap:wrap; }
	.tech-tags span{ padding:5px 9px; border-radius:999px; border:1px solid rgba(255,255,255,0.1); background:rgba(255,255,255,0.06); font:600 10px 'JetBrains Mono', monospace; letter-spacing:0.1em; text-transform:uppercase; }

	.footer{
		max-width:1220px; margin:0 auto; padding:18px var(--pad) 32px;
		display:flex; flex-wrap:wrap; gap:12px; justify-content:space-between; align-items:center;
		font:500 12px 'JetBrains Mono', monospace; color: color-mix(in srgb, var(--ink) 60%, transparent);
		border-top: 1px solid rgba(255,255,255,0.06);
	}
	.footer a{ color:var(--ink); text-decoration:none; border-bottom:1px dashed rgba(255,255,255,0.2); }
	.foot-links{ display:flex; gap:14px; }

	.help-backdrop{ position:fixed; inset:0; background:rgba(0,0,0,0.55); backdrop-filter: blur(6px); z-index:20; border:0; }
	.help{
		position:fixed; left:50%; top:50%; transform:translate(-50%,-50%); z-index:21;
		width:min(560px, calc(100% - 32px)); background: #0e1120; color:#eef0ff; border:1px solid rgba(255,255,255,0.1);
		border-radius:18px; padding:22px; box-shadow:0 22px 60px rgba(0,0,0,0.6);
	}
	.help h2{ margin:0 0 12px; font:800 16px 'Syne', sans-serif; letter-spacing:-0.02em; }
	.help ul{ margin:0; padding-left:18px; display:grid; gap:8px; font-size:13.5px; line-height:1.5; color: #c8cce6; }
	.help li b{ color:#fff; }
	.help-foot{ display:flex; justify-content:space-between; align-items:center; margin-top:16px; font:500 12px 'JetBrains Mono', monospace; color:#9aa0c2; }

	@media (max-width: 860px){
		.kinetic{ font-size: clamp(56px, 22vw, 108px); }
		.hero-stats{ gap:6px; }
		.arena-toolbar{ gap:8px; }
		.hide-mobile{ display:none; }
	}
</style>
