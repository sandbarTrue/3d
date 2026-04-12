# Procedural 3D Robot Design for Discover Page

Research for building 4 distinct robot styles using only Three.js built-in geometries. All robots fit within a ~2x2x2 unit cube. Uses the project's existing `createEditorMaterial()` cosine-palette ShaderMaterial for body parts and `MeshBasicMaterial` for emissive accents (eyes, lights).

---

## 1. Group Hierarchy Pattern

Every robot uses nested `THREE.Group` for articulated joints. This allows independent rotation of head, arms, legs without affecting siblings.

```
robotGroup (root — positioned at origin, used for float/bob)
├── body (torso mesh + detail meshes)
├── headJoint (Group — pivot at neck)
│   └── head (mesh + eyes + antenna)
├── leftArmJoint (Group — pivot at shoulder)
│   ├── upperArm (mesh)
│   └── lowerArmJoint (Group — pivot at elbow)
│       └── forearm/hand (mesh)
├── rightArmJoint (same structure)
├── leftLegJoint (Group — pivot at hip)
│   └── leg (mesh)
└── rightLegJoint (same structure)
```

Key code pattern for joint creation:

```js
function makeJoint(parent, pivotPos) {
  const joint = new THREE.Group();
  joint.position.copy(pivotPos);
  parent.add(joint);
  return joint;
}

// Example: head joint at top of torso
const headJoint = makeJoint(robotGroup, new THREE.Vector3(0, 0.65, 0));
// Meshes added to headJoint are positioned relative to the joint
const headMesh = new THREE.Mesh(headGeo, headMat);
headMesh.position.set(0, 0.25, 0); // offset from joint pivot
headJoint.add(headMesh);

// To animate: headJoint.rotation.y = Math.sin(t * 0.5) * 0.3;
```

---

## 2. Four Robot Styles

### Style A — Cute/Round (EVE-like)

**Palette suggestion:** `porcelain` or `mint`

**Geometry breakdown:**
| Part | Geometry | Parameters | Position (relative to parent) |
|------|----------|-----------|-------------------------------|
| Body (torso) | CapsuleGeometry | (0.32, 0.4, 8, 16) | (0, 0, 0) |
| Head | SphereGeometry | (0.28, 24, 24) | (0, 0.22, 0) relative to headJoint |
| Left Eye | SphereGeometry | (0.06, 12, 12) | (-0.1, 0.06, 0.24) relative to head |
| Right Eye | SphereGeometry | (0.06, 12, 12) | (0.1, 0.06, 0.24) relative to head |
| Eye Pupil (x2) | SphereGeometry | (0.03, 8, 8) | (0, 0, 0.055) relative to each eye |
| Left Arm | CapsuleGeometry | (0.08, 0.25, 4, 8) | (0, -0.15, 0) relative to armJoint |
| Right Arm | CapsuleGeometry | (0.08, 0.25, 4, 8) | (0, -0.15, 0) relative to armJoint |
| Left Foot | SphereGeometry | (0.1, 12, 12) scaled (1, 0.6, 1.3) | (-0.15, -0.55, 0.03) |
| Right Foot | SphereGeometry | (0.1, 12, 12) scaled (1, 0.6, 1.3) | (0.15, -0.55, 0.03) |
| Antenna base | CylinderGeometry | (0.015, 0.015, 0.12, 6) | (0, 0.28, 0) relative to head |
| Antenna tip | SphereGeometry | (0.03, 8, 8) | (0, 0.35, 0) relative to head |

**Joint positions:**
- headJoint: (0, 0.45, 0)
- leftArmJoint: (-0.35, 0.2, 0)
- rightArmJoint: (0.35, 0.2, 0)

**Materials:**
- Body/head/arms/feet: `createEditorMaterial('porcelain')` or `createEditorMaterial('mint')`
- Eyes: `new THREE.MeshBasicMaterial({ color: 0x44ffff })` (cyan glow)
- Pupils: `new THREE.MeshBasicMaterial({ color: 0x111122 })`
- Antenna tip: `new THREE.MeshBasicMaterial({ color: 0xff6688 })` (pink glow)

**Build code:**

```js
function buildCuteRobot(palette) {
  const root = new THREE.Group();
  const mat = createEditorMaterial(palette || 'porcelain');
  const eyeMat = new THREE.MeshBasicMaterial({ color: 0x44ffff });
  const pupilMat = new THREE.MeshBasicMaterial({ color: 0x111122 });
  const glowMat = new THREE.MeshBasicMaterial({ color: 0xff6688 });

  // Body
  const body = new THREE.Mesh(new THREE.CapsuleGeometry(0.32, 0.4, 8, 16), mat);
  root.add(body);

  // Head joint + head
  const headJoint = new THREE.Group();
  headJoint.position.set(0, 0.45, 0);
  root.add(headJoint);

  const head = new THREE.Mesh(new THREE.SphereGeometry(0.28, 24, 24), mat);
  head.position.set(0, 0.22, 0);
  headJoint.add(head);

  // Eyes
  const eyeGeo = new THREE.SphereGeometry(0.06, 12, 12);
  const leftEye = new THREE.Mesh(eyeGeo, eyeMat);
  leftEye.position.set(-0.1, 0.06, 0.24);
  headJoint.add(leftEye);
  const rightEye = new THREE.Mesh(eyeGeo, eyeMat);
  rightEye.position.set(0.1, 0.06, 0.24);
  headJoint.add(rightEye);

  // Pupils
  const pupilGeo = new THREE.SphereGeometry(0.03, 8, 8);
  const lPupil = new THREE.Mesh(pupilGeo, pupilMat);
  lPupil.position.set(0, 0, 0.055);
  leftEye.add(lPupil);
  const rPupil = new THREE.Mesh(pupilGeo, pupilMat);
  rPupil.position.set(0, 0, 0.055);
  rightEye.add(rPupil);

  // Antenna
  const antBase = new THREE.Mesh(new THREE.CylinderGeometry(0.015, 0.015, 0.12, 6), mat);
  antBase.position.set(0, 0.5, 0);
  headJoint.add(antBase);
  const antTip = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 8), glowMat);
  antTip.position.set(0, 0.57, 0);
  headJoint.add(antTip);

  // Arms
  const armGeo = new THREE.CapsuleGeometry(0.08, 0.25, 4, 8);
  const leftArmJoint = new THREE.Group();
  leftArmJoint.position.set(-0.35, 0.2, 0);
  root.add(leftArmJoint);
  const lArm = new THREE.Mesh(armGeo, mat);
  lArm.position.set(0, -0.15, 0);
  leftArmJoint.add(lArm);

  const rightArmJoint = new THREE.Group();
  rightArmJoint.position.set(0.35, 0.2, 0);
  root.add(rightArmJoint);
  const rArm = new THREE.Mesh(armGeo, mat);
  rArm.position.set(0, -0.15, 0);
  rightArmJoint.add(rArm);

  // Feet (no leg joints — stubby)
  const footGeo = new THREE.SphereGeometry(0.1, 12, 12);
  const lFoot = new THREE.Mesh(footGeo, mat);
  lFoot.position.set(-0.15, -0.55, 0.03);
  lFoot.scale.set(1, 0.6, 1.3);
  root.add(lFoot);
  const rFoot = new THREE.Mesh(footGeo, mat);
  rFoot.position.set(0.15, -0.55, 0.03);
  rFoot.scale.set(1, 0.6, 1.3);
  root.add(rFoot);

  return {
    root,
    headJoint,
    leftArmJoint,
    rightArmJoint,
    eyes: [leftEye, rightEye],
    antTip,
    body,
    shaderMats: [mat],
  };
}
```

---

### Style B — Mech/Angular

**Palette suggestion:** `ocean` or `stone`

**Geometry breakdown:**
| Part | Geometry | Parameters | Position |
|------|----------|-----------|----------|
| Torso | BoxGeometry | (0.5, 0.55, 0.35) | (0, 0, 0) |
| Chest plate | BoxGeometry | (0.42, 0.2, 0.05) | (0, 0.08, 0.19) |
| Head | BoxGeometry | (0.28, 0.22, 0.26) | (0, 0.16, 0) relative to headJoint |
| Visor | BoxGeometry | (0.24, 0.06, 0.05) | (0, 0.04, 0.135) relative to head |
| Shoulder pad L | BoxGeometry | (0.2, 0.1, 0.2) | (-0.05, 0.05, 0) relative to armJoint |
| Shoulder pad R | BoxGeometry | (0.2, 0.1, 0.2) | (0.05, 0.05, 0) relative to armJoint |
| Upper arm | BoxGeometry | (0.12, 0.28, 0.12) | (0, -0.2, 0) relative to armJoint |
| Forearm | BoxGeometry | (0.1, 0.22, 0.1) | (0, -0.18, 0) relative to elbowJoint |
| Hip block | BoxGeometry | (0.44, 0.12, 0.3) | (0, -0.34, 0) |
| Upper leg | BoxGeometry | (0.13, 0.25, 0.13) | (0, -0.16, 0) relative to legJoint |
| Lower leg | BoxGeometry | (0.11, 0.22, 0.11) | (0, -0.14, 0) relative to kneeJoint |
| Foot | BoxGeometry | (0.15, 0.06, 0.22) | (0, -0.14, 0.03) relative to kneeJoint |

**Joint positions:**
- headJoint: (0, 0.35, 0)
- leftArmJoint: (-0.38, 0.18, 0)
- rightArmJoint: (0.38, 0.18, 0)
- leftElbowJoint: (0, -0.36, 0) relative to armJoint
- rightElbowJoint: same
- leftLegJoint: (-0.14, -0.42, 0)
- rightLegJoint: (0.14, -0.42, 0)
- kneeJoints: (0, -0.28, 0) relative to legJoint

**Materials:**
- Body/limbs: `createEditorMaterial('ocean')` or `createEditorMaterial('stone')`
- Visor: `new THREE.MeshBasicMaterial({ color: 0x00aaff, transparent: true, opacity: 0.85 })`
- Chest plate accents: `new THREE.MeshBasicMaterial({ color: 0xff4444 })`

**Build code:**

```js
function buildMechRobot(palette) {
  const root = new THREE.Group();
  const mat = createEditorMaterial(palette || 'ocean');
  const visorMat = new THREE.MeshBasicMaterial({ color: 0x00aaff, transparent: true, opacity: 0.85 });
  const accentMat = new THREE.MeshBasicMaterial({ color: 0xff4444 });

  // Torso
  const torso = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.55, 0.35), mat);
  root.add(torso);

  // Chest plate
  const chest = new THREE.Mesh(new THREE.BoxGeometry(0.42, 0.2, 0.05), accentMat);
  chest.position.set(0, 0.08, 0.19);
  root.add(chest);

  // Head
  const headJoint = new THREE.Group();
  headJoint.position.set(0, 0.35, 0);
  root.add(headJoint);
  const head = new THREE.Mesh(new THREE.BoxGeometry(0.28, 0.22, 0.26), mat);
  head.position.set(0, 0.16, 0);
  headJoint.add(head);

  // Visor
  const visor = new THREE.Mesh(new THREE.BoxGeometry(0.24, 0.06, 0.05), visorMat);
  visor.position.set(0, 0.14, 0.135);
  headJoint.add(visor);

  // Arms with shoulders + elbows
  function buildArm(side) {
    const sign = side === 'left' ? -1 : 1;
    const armJoint = new THREE.Group();
    armJoint.position.set(sign * 0.38, 0.18, 0);
    root.add(armJoint);

    // Shoulder pad
    const shoulder = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.1, 0.2), mat);
    shoulder.position.set(sign * -0.05, 0.05, 0);
    armJoint.add(shoulder);

    // Upper arm
    const upper = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.28, 0.12), mat);
    upper.position.set(0, -0.2, 0);
    armJoint.add(upper);

    // Elbow joint
    const elbowJoint = new THREE.Group();
    elbowJoint.position.set(0, -0.36, 0);
    armJoint.add(elbowJoint);

    const forearm = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.22, 0.1), mat);
    forearm.position.set(0, -0.13, 0);
    elbowJoint.add(forearm);

    return { armJoint, elbowJoint };
  }

  const leftArm = buildArm('left');
  const rightArm = buildArm('right');

  // Hip
  const hip = new THREE.Mesh(new THREE.BoxGeometry(0.44, 0.12, 0.3), mat);
  hip.position.set(0, -0.34, 0);
  root.add(hip);

  // Legs
  function buildLeg(side) {
    const sign = side === 'left' ? -1 : 1;
    const legJoint = new THREE.Group();
    legJoint.position.set(sign * 0.14, -0.42, 0);
    root.add(legJoint);

    const upper = new THREE.Mesh(new THREE.BoxGeometry(0.13, 0.25, 0.13), mat);
    upper.position.set(0, -0.16, 0);
    legJoint.add(upper);

    const kneeJoint = new THREE.Group();
    kneeJoint.position.set(0, -0.28, 0);
    legJoint.add(kneeJoint);

    const lower = new THREE.Mesh(new THREE.BoxGeometry(0.11, 0.22, 0.11), mat);
    lower.position.set(0, -0.14, 0);
    kneeJoint.add(lower);

    const foot = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.06, 0.22), mat);
    foot.position.set(0, -0.28, 0.03);
    kneeJoint.add(foot);

    return { legJoint, kneeJoint };
  }

  const leftLeg = buildLeg('left');
  const rightLeg = buildLeg('right');

  return {
    root,
    headJoint,
    visor,
    leftArm, rightArm,
    leftLeg, rightLeg,
    torso,
    shaderMats: [mat],
  };
}
```

---

### Style C — Retro/Boxy (Classic Tin Robot)

**Palette suggestion:** `ember` or `coral`

**Geometry breakdown:**
| Part | Geometry | Parameters | Position |
|------|----------|-----------|----------|
| Body | BoxGeometry | (0.5, 0.6, 0.35) | (0, 0, 0) |
| Belly dial | CylinderGeometry | (0.06, 0.06, 0.04, 12) rotated X=PI/2 | (0, -0.05, 0.18) |
| Chest buttons (x3) | CylinderGeometry | (0.03, 0.03, 0.04, 8) rotated X=PI/2 | stacked vertically on chest |
| Head | BoxGeometry | (0.36, 0.3, 0.3) | (0, 0.2, 0) relative to headJoint |
| Left eye | CylinderGeometry | (0.05, 0.05, 0.04, 12) rotated X=PI/2 | (-0.08, 0.08, 0.155) |
| Right eye | same | (0.08, 0.08, 0.155) |
| Mouth grill | BoxGeometry | (0.16, 0.04, 0.04) (x3 stacked) | centered on face, y=-0.03, -0.01, 0.01 |
| Antenna L | CylinderGeometry | (0.012, 0.012, 0.18, 6) | (-0.1, 0.4, 0) relative to head |
| Antenna R | same | (0.1, 0.4, 0) |
| Antenna tip L/R | SphereGeometry | (0.025, 8, 8) | top of each antenna |
| Upper arm | BoxGeometry | (0.12, 0.3, 0.12) | (0, -0.2, 0) relative to armJoint |
| Hand | BoxGeometry | (0.1, 0.1, 0.1) | (0, -0.4, 0) relative to armJoint |
| Upper leg | BoxGeometry | (0.14, 0.3, 0.14) | (0, -0.18, 0) relative to legJoint |
| Foot | BoxGeometry | (0.18, 0.06, 0.24) | (0, -0.36, 0.03) |

**Joint positions:**
- headJoint: (0, 0.38, 0)
- leftArmJoint: (-0.36, 0.15, 0)
- rightArmJoint: (0.36, 0.15, 0)
- leftLegJoint: (-0.14, -0.36, 0)
- rightLegJoint: (0.14, -0.36, 0)

**Materials:**
- Body/head/limbs: `createEditorMaterial('ember')`
- Eyes: `new THREE.MeshBasicMaterial({ color: 0xffdd00 })` (yellow glow)
- Belly dial: `new THREE.MeshBasicMaterial({ color: 0xff3333 })`
- Chest buttons: `new THREE.MeshBasicMaterial({ color: 0x33ff33 })` / `0xff3333` / `0x3399ff`
- Antenna tips: `new THREE.MeshBasicMaterial({ color: 0xff4444 })`
- Mouth grill: use darker variant of body mat or `new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.8 })`

**Build code:**

```js
function buildRetroRobot(palette) {
  const root = new THREE.Group();
  const mat = createEditorMaterial(palette || 'ember');
  const eyeMat = new THREE.MeshBasicMaterial({ color: 0xffdd00 });
  const dialMat = new THREE.MeshBasicMaterial({ color: 0xff3333 });
  const grillMat = new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.8, roughness: 0.3 });
  const btnColors = [0x33ff33, 0xff3333, 0x3399ff];

  // Body
  const body = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.6, 0.35), mat);
  root.add(body);

  // Belly dial
  const dial = new THREE.Mesh(new THREE.CylinderGeometry(0.06, 0.06, 0.04, 12), dialMat);
  dial.rotation.x = Math.PI / 2;
  dial.position.set(0, -0.05, 0.19);
  root.add(dial);

  // Chest buttons
  for (let i = 0; i < 3; i++) {
    const btn = new THREE.Mesh(
      new THREE.CylinderGeometry(0.025, 0.025, 0.04, 8),
      new THREE.MeshBasicMaterial({ color: btnColors[i] })
    );
    btn.rotation.x = Math.PI / 2;
    btn.position.set(-0.08 + i * 0.08, 0.15, 0.19);
    root.add(btn);
  }

  // Head
  const headJoint = new THREE.Group();
  headJoint.position.set(0, 0.38, 0);
  root.add(headJoint);
  const head = new THREE.Mesh(new THREE.BoxGeometry(0.36, 0.3, 0.3), mat);
  head.position.set(0, 0.2, 0);
  headJoint.add(head);

  // Eyes
  const eyeGeo = new THREE.CylinderGeometry(0.05, 0.05, 0.04, 12);
  const lEye = new THREE.Mesh(eyeGeo, eyeMat);
  lEye.rotation.x = Math.PI / 2;
  lEye.position.set(-0.08, 0.24, 0.16);
  headJoint.add(lEye);
  const rEye = new THREE.Mesh(eyeGeo, eyeMat);
  rEye.rotation.x = Math.PI / 2;
  rEye.position.set(0.08, 0.24, 0.16);
  headJoint.add(rEye);

  // Mouth grill (3 horizontal bars)
  for (let i = 0; i < 3; i++) {
    const bar = new THREE.Mesh(new THREE.BoxGeometry(0.16, 0.02, 0.04), grillMat);
    bar.position.set(0, 0.11 + i * 0.03, 0.16);
    headJoint.add(bar);
  }

  // Antennas
  const antGeo = new THREE.CylinderGeometry(0.012, 0.012, 0.18, 6);
  const antTipGeo = new THREE.SphereGeometry(0.025, 8, 8);
  const antTipMat = new THREE.MeshBasicMaterial({ color: 0xff4444 });
  [-0.1, 0.1].forEach(x => {
    const ant = new THREE.Mesh(antGeo, mat);
    ant.position.set(x, 0.44, 0);
    headJoint.add(ant);
    const tip = new THREE.Mesh(antTipGeo, antTipMat);
    tip.position.set(x, 0.54, 0);
    headJoint.add(tip);
  });

  // Arms
  const armGeo = new THREE.BoxGeometry(0.12, 0.3, 0.12);
  const handGeo = new THREE.BoxGeometry(0.1, 0.1, 0.1);
  function buildArm(sign) {
    const joint = new THREE.Group();
    joint.position.set(sign * 0.36, 0.15, 0);
    root.add(joint);
    const arm = new THREE.Mesh(armGeo, mat);
    arm.position.set(0, -0.2, 0);
    joint.add(arm);
    const hand = new THREE.Mesh(handGeo, mat);
    hand.position.set(0, -0.4, 0);
    joint.add(hand);
    return joint;
  }
  const leftArmJoint = buildArm(-1);
  const rightArmJoint = buildArm(1);

  // Legs
  const legGeo = new THREE.BoxGeometry(0.14, 0.3, 0.14);
  const footGeo = new THREE.BoxGeometry(0.18, 0.06, 0.24);
  function buildLeg(sign) {
    const joint = new THREE.Group();
    joint.position.set(sign * 0.14, -0.36, 0);
    root.add(joint);
    const leg = new THREE.Mesh(legGeo, mat);
    leg.position.set(0, -0.18, 0);
    joint.add(leg);
    const foot = new THREE.Mesh(footGeo, mat);
    foot.position.set(0, -0.36, 0.03);
    joint.add(foot);
    return joint;
  }
  const leftLegJoint = buildLeg(-1);
  const rightLegJoint = buildLeg(1);

  return {
    root,
    headJoint,
    leftArmJoint, rightArmJoint,
    leftLegJoint, rightLegJoint,
    eyes: [lEye, rEye],
    body,
    shaderMats: [mat],
  };
}
```

---

### Style D — Drone/Spider

**Palette suggestion:** `hologram` or `aurora`

**Geometry breakdown:**
| Part | Geometry | Parameters | Position |
|------|----------|-----------|----------|
| Central body | SphereGeometry | (0.3, 24, 24) | (0, 0, 0) |
| Eye lens | SphereGeometry | (0.12, 16, 16) | (0, 0.05, 0.28) |
| Eye inner | SphereGeometry | (0.06, 12, 12) | (0, 0, 0.06) relative to lens |
| Top sensor | CylinderGeometry | (0.02, 0.04, 0.12, 8) | (0, 0.32, 0) |
| Sensor light | SphereGeometry | (0.03, 8, 8) | (0, 0.39, 0) |
| Leg upper (x6) | CylinderGeometry | (0.025, 0.018, 0.45, 6) | angled outward from body |
| Leg lower (x6) | CylinderGeometry | (0.018, 0.012, 0.35, 6) | angled downward from knee |
| Knee joint (x6) | SphereGeometry | (0.03, 6, 6) | at connection between upper/lower |
| Foot tip (x6) | SphereGeometry | (0.02, 6, 6) | at end of lower leg |

**Leg placement pattern (6 legs, evenly spaced around body):**

```js
const LEG_COUNT = 6;
for (let i = 0; i < LEG_COUNT; i++) {
  const angle = (i / LEG_COUNT) * Math.PI * 2;
  const legJoint = new THREE.Group();
  // Position at body surface
  legJoint.position.set(
    Math.cos(angle) * 0.25,
    -0.05,
    Math.sin(angle) * 0.25
  );
  // Rotate to face outward
  legJoint.rotation.y = -angle;
  // Tilt outward and down
  legJoint.rotation.z = 0.6; // splay angle
  root.add(legJoint);

  // Upper leg segment
  const upper = new THREE.Mesh(upperLegGeo, mat);
  upper.position.set(0, -0.22, 0);
  legJoint.add(upper);

  // Knee joint (for animation)
  const kneeJoint = new THREE.Group();
  kneeJoint.position.set(0, -0.45, 0);
  legJoint.add(kneeJoint);

  // Knee sphere
  const knee = new THREE.Mesh(kneeGeo, mat);
  kneeJoint.add(knee);

  // Lower leg — angled further down
  const lower = new THREE.Mesh(lowerLegGeo, mat);
  lower.position.set(0, -0.18, 0);
  lower.rotation.z = -0.8; // bend back toward ground
  kneeJoint.add(lower);

  // Foot tip glow
  const tip = new THREE.Mesh(footGeo, glowMat);
  tip.position.set(0, -0.35, 0);
  kneeJoint.add(tip);
}
```

**Materials:**
- Body/legs: `createEditorMaterial('hologram')`
- Eye lens: `new THREE.MeshBasicMaterial({ color: 0xff2244, transparent: true, opacity: 0.8 })`
- Eye inner: `new THREE.MeshBasicMaterial({ color: 0xff0000 })`
- Sensor/foot tips: `new THREE.MeshBasicMaterial({ color: 0x44ffaa })`

**Build code:**

```js
function buildDroneRobot(palette) {
  const root = new THREE.Group();
  const mat = createEditorMaterial(palette || 'hologram');
  const eyeMat = new THREE.MeshBasicMaterial({ color: 0xff2244, transparent: true, opacity: 0.8 });
  const eyeInnerMat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  const glowMat = new THREE.MeshBasicMaterial({ color: 0x44ffaa });

  // Central body
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.3, 24, 24), mat);
  root.add(body);

  // Eye
  const eyeLens = new THREE.Mesh(new THREE.SphereGeometry(0.12, 16, 16), eyeMat);
  eyeLens.position.set(0, 0.05, 0.28);
  root.add(eyeLens);
  const eyeInner = new THREE.Mesh(new THREE.SphereGeometry(0.06, 12, 12), eyeInnerMat);
  eyeInner.position.set(0, 0, 0.06);
  eyeLens.add(eyeInner);

  // Top sensor
  const sensor = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.04, 0.12, 8), mat);
  sensor.position.set(0, 0.32, 0);
  root.add(sensor);
  const sensorLight = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 8), glowMat);
  sensorLight.position.set(0, 0.39, 0);
  root.add(sensorLight);

  // Legs
  const LEG_COUNT = 6;
  const upperGeo = new THREE.CylinderGeometry(0.025, 0.018, 0.45, 6);
  const lowerGeo = new THREE.CylinderGeometry(0.018, 0.012, 0.35, 6);
  const kneeGeo = new THREE.SphereGeometry(0.03, 6, 6);
  const tipGeo = new THREE.SphereGeometry(0.02, 6, 6);
  const legs = [];

  for (let i = 0; i < LEG_COUNT; i++) {
    const angle = (i / LEG_COUNT) * Math.PI * 2;
    const legJoint = new THREE.Group();
    legJoint.position.set(
      Math.cos(angle) * 0.25,
      -0.05,
      Math.sin(angle) * 0.25
    );
    legJoint.rotation.y = -angle;
    legJoint.rotation.z = 0.6;
    root.add(legJoint);

    const upper = new THREE.Mesh(upperGeo, mat);
    upper.position.set(0, -0.22, 0);
    legJoint.add(upper);

    const kneeJoint = new THREE.Group();
    kneeJoint.position.set(0, -0.45, 0);
    legJoint.add(kneeJoint);

    const knee = new THREE.Mesh(kneeGeo, mat);
    kneeJoint.add(knee);

    const lower = new THREE.Mesh(lowerGeo, mat);
    lower.position.set(0, -0.18, 0);
    lower.rotation.z = -0.8;
    kneeJoint.add(lower);

    const tip = new THREE.Mesh(tipGeo, glowMat);
    tip.position.set(0, -0.35, 0);
    kneeJoint.add(tip);

    legs.push({ legJoint, kneeJoint });
  }

  return {
    root,
    body,
    eyeLens,
    eyeInner,
    sensorLight,
    legs,
    shaderMats: [mat],
  };
}
```

---

## 3. Materials Strategy

### Body materials — use existing ShaderMaterial system

The project already has `createEditorMaterial(paletteKey)` which produces a `ShaderMaterial` with:
- Cosine color palette (`uPalA/B/C/D`)
- Fresnel rim lighting (`uRim1/uRim2`, `uFresnelStr/Pow`)
- Fake environment reflection
- `uTime` uniform for animated color shifting
- `uHover` uniform for hover glow

This is the primary material for all robot body parts. A single material instance can be shared across all meshes of one robot (since it's the same palette).

**Recommended palette assignments:**
| Robot Style | Primary Palette | Alt Palette |
|-------------|----------------|-------------|
| Cute/Round | `porcelain` | `mint` |
| Mech/Angular | `ocean` | `stone` |
| Retro/Boxy | `ember` | `coral` |
| Drone/Spider | `hologram` | `aurora` |

### Emissive accents — MeshBasicMaterial

For eyes, indicator lights, antenna tips, visors — use `MeshBasicMaterial`. It renders at full brightness regardless of scene lighting, creating a "glowing" effect. No need for actual emissive maps or post-processing bloom.

```js
// Glowing eye
const eyeMat = new THREE.MeshBasicMaterial({ color: 0x44ffff });

// Semi-transparent visor
const visorMat = new THREE.MeshBasicMaterial({
  color: 0x00aaff,
  transparent: true,
  opacity: 0.85,
});

// Pulsing glow — animate in the loop:
eyeMat.color.setHSL(0.5, 1.0, 0.5 + Math.sin(t * 3) * 0.15);
```

---

## 4. Animation System

All animations designed for small card format — subtle, looping, no sudden movements. Each robot builder returns joint references so the animation function can manipulate them.

### Animation function pattern

```js
function createRobotAnimFn(robot, style) {
  let lastT = 0;
  let curHover = 0;

  return (t, hover) => {
    const dt = Math.min(t - lastT, 0.05);
    lastT = t;

    // Smooth hover interpolation
    curHover += ((hover ? 1 : 0) - curHover) * 0.08;

    // ── Float/bob (all styles) ──
    robot.root.position.y = Math.sin(t * 1.2) * 0.06;

    // ── Head turn (all styles that have headJoint) ──
    if (robot.headJoint) {
      robot.headJoint.rotation.y = Math.sin(t * 0.5) * 0.25;
      robot.headJoint.rotation.x = Math.sin(t * 0.7) * 0.05;
    }

    // ── Breathing (torso scale pulse) ──
    if (robot.body) {
      const breath = 1.0 + Math.sin(t * 1.5) * 0.015;
      robot.body.scale.set(1, breath, 1);
    }

    // ── Eye glow pulse ──
    if (robot.eyes) {
      const eyeBright = 0.5 + Math.sin(t * 3.0) * 0.2 + curHover * 0.3;
      robot.eyes.forEach(eye => {
        eye.material.color.setHSL(
          eye.material.color.getHSL({}).h,
          1.0,
          eyeBright
        );
      });
    }

    // ── Style-specific animations ──
    switch (style) {
      case 'cute':
        // Gentle arm sway
        if (robot.leftArmJoint) {
          robot.leftArmJoint.rotation.z = Math.sin(t * 0.8) * 0.15;
          robot.rightArmJoint.rotation.z = Math.sin(t * 0.8 + Math.PI) * 0.15;
        }
        // Antenna tip glow pulse
        if (robot.antTip) {
          robot.antTip.material.color.setHSL(0.95, 1, 0.5 + Math.sin(t * 4) * 0.3);
        }
        break;

      case 'mech':
        // Stiff arm idle — slight sway
        if (robot.leftArm) {
          robot.leftArm.armJoint.rotation.x = Math.sin(t * 0.4) * 0.08;
          robot.rightArm.armJoint.rotation.x = Math.sin(t * 0.4 + 1) * 0.08;
          // Elbow slight bend
          robot.leftArm.elbowJoint.rotation.x = -0.15 + Math.sin(t * 0.6) * 0.05;
          robot.rightArm.elbowJoint.rotation.x = -0.15 + Math.sin(t * 0.6 + 1) * 0.05;
        }
        // Visor pulse
        if (robot.visor) {
          const visorBright = 0.5 + Math.sin(t * 2) * 0.15 + curHover * 0.35;
          robot.visor.material.opacity = 0.6 + visorBright * 0.3;
        }
        // Walking leg idle
        if (robot.leftLeg) {
          robot.leftLeg.legJoint.rotation.x = Math.sin(t * 0.6) * 0.06;
          robot.rightLeg.legJoint.rotation.x = Math.sin(t * 0.6 + Math.PI) * 0.06;
        }
        break;

      case 'retro':
        // Arm wave (one arm waves, other hangs)
        if (robot.rightArmJoint) {
          robot.rightArmJoint.rotation.z = -0.3 + Math.sin(t * 1.5) * 0.4;
          robot.leftArmJoint.rotation.z = 0.1 + Math.sin(t * 0.5) * 0.05;
        }
        // Stiff walk
        if (robot.leftLegJoint) {
          robot.leftLegJoint.rotation.x = Math.sin(t * 0.8) * 0.1;
          robot.rightLegJoint.rotation.x = Math.sin(t * 0.8 + Math.PI) * 0.1;
        }
        break;

      case 'drone':
        // Body gentle rotation
        robot.root.rotation.y = t * 0.3;
        // Leg crawl — each leg phases differently
        if (robot.legs) {
          robot.legs.forEach((leg, i) => {
            const phase = (i / robot.legs.length) * Math.PI * 2;
            // Subtle leg "stepping" animation
            leg.kneeJoint.rotation.z = -0.8 + Math.sin(t * 1.2 + phase) * 0.15;
            leg.legJoint.rotation.z = 0.6 + Math.sin(t * 0.8 + phase) * 0.1;
          });
        }
        // Eye look around
        if (robot.eyeLens) {
          robot.eyeLens.position.x = Math.sin(t * 0.7) * 0.03;
          robot.eyeLens.position.y = 0.05 + Math.cos(t * 0.5) * 0.02;
        }
        // Sensor blink
        if (robot.sensorLight) {
          robot.sensorLight.material.color.setHSL(
            0.4, 1.0, 0.3 + Math.sin(t * 5) * 0.5
          );
        }
        break;
    }

    // ── Hover: speed up rotation ──
    if (curHover > 0.01) {
      robot.root.rotation.y += curHover * 0.5 * dt;
    }

    // ── Update shader time ──
    robot.shaderMats.forEach(m => {
      m.uniforms.uTime.value = t;
      m.uniforms.uHover.value += ((hover ? 1 : 0) - m.uniforms.uHover.value) * 0.08;
    });
  };
}
```

### Animation details per style

| Animation | Cute | Mech | Retro | Drone |
|-----------|------|------|-------|-------|
| Float/bob | Y: sin(t*1.2)*0.06 | Y: sin(t*1.2)*0.04 | Y: sin(t*1.0)*0.05 | Y: sin(t*1.5)*0.08 |
| Head turn | Y: sin(t*0.5)*0.25 | Y: sin(t*0.5)*0.15 | Y: sin(t*0.4)*0.2 | N/A (body rotates) |
| Breathing | scaleY: 1+sin(t*1.5)*0.015 | scaleY: 1+sin(t*1.5)*0.01 | scaleY: 1+sin(t*1.2)*0.01 | N/A |
| Eye glow | HSL lightness pulse | Visor opacity pulse | HSL lightness pulse | Eye position wander |
| Arms | Gentle sway z | Stiff sway x | Wave (one arm) | N/A |
| Legs | None (stubby) | Subtle walk x | Stiff walk x | Crawl phase offset |
| Special | Antenna tip HSL | Elbow bend | — | Body Y rotation, sensor blink |
| Hover boost | Faster float + rotation | Faster float + rotation | Faster float + rotation | Faster spin + leg speed |

---

## 5. Camera and Scale

Robots should fit in roughly a 2x2x2 unit cube. With the card's existing camera setup:

```js
camera.position.set(0, 0.2, 3.5);
camera.lookAt(0, 0, 0);
```

- FOV 35 (already used by existing cards)
- Camera at z=3.5 to z=4 gives good framing for a 2-unit-tall robot
- Slight y offset (0.2) puts camera at "chest level", looking slightly up at the face
- The robot `root` group sits at origin; bob animation moves it +-0.06 on Y

For the drone/spider (shorter, wider), use:
```js
camera.position.set(0, 0.5, 3.2);
camera.lookAt(0, 0, 0);
```

---

## 6. Integration with Existing Card System

### Card definition pattern

```js
// In the cards array:
{ type: '3d-robot', height: 280, bg: '#0e0d16',
  robotStyle: 'cute', palette: 'porcelain',
  foot: { color: '#e8e0d8', name: 'Robot Lab', likes: '3.2k' } },

{ type: '3d-robot', height: 300, bg: '#0a0e18',
  robotStyle: 'mech', palette: 'ocean',
  foot: { color: '#74b9ff', name: 'Mech Works', likes: '2.8k' } },

{ type: '3d-robot', height: 280, bg: '#1a0e08',
  robotStyle: 'retro', palette: 'ember',
  foot: { color: '#fab1a0', name: 'Retro Factory', likes: '2.1k' } },

{ type: '3d-robot', height: 260, bg: '#12101e',
  robotStyle: 'drone', palette: 'hologram',
  foot: { color: '#a29bfe', name: 'Drone Lab', likes: '1.9k' } },
```

### Scene creation integration

Add to `createMiniScene()`:

```js
if (sceneType === '3d-robot') {
  // Camera setup
  const camY = config.robotStyle === 'drone' ? 0.5 : 0.2;
  const camZ = config.robotStyle === 'drone' ? 3.2 : 3.5;
  camera.position.set(0, camY, camZ);
  camera.lookAt(0, 0, 0);

  // Build robot
  let robot;
  switch (config.robotStyle) {
    case 'cute':  robot = buildCuteRobot(config.palette); break;
    case 'mech':  robot = buildMechRobot(config.palette); break;
    case 'retro': robot = buildRetroRobot(config.palette); break;
    case 'drone': robot = buildDroneRobot(config.palette); break;
  }

  group.add(robot.root);
  shaderMats.push(...robot.shaderMats);

  // Create the animation function
  const robotAnim = createRobotAnimFn(robot, config.robotStyle);
  animFn = (t, hover) => robotAnim(t, hover);
}
```

### Badge and label

Re-use the existing badge/label pattern from `3d-shape` cards:
```js
if (config.type === '3d-robot') {
  card.classList.add('card-clickable');
  const badge = document.createElement('div');
  badge.className = 'badge-3d';
  badge.textContent = 'ROBOT';
  sceneDiv.appendChild(badge);
  const label = document.createElement('div');
  label.className = 'shape-label';
  label.textContent = `${styleName} · ${palName}`;
  sceneDiv.appendChild(label);
}
```

---

## 7. Performance Notes

- **Shared material instances**: Each robot gets ONE `ShaderMaterial` instance shared across all body meshes. Only emissive accents get separate simple materials.
- **Geometry reuse**: Arm geometries are created once and reused for left/right. Same for legs.
- **Low poly counts**: Most geometries use low segment counts (8-24 segments). Total triangle count per robot: roughly 2,000-4,000.
- **Scissor rendering**: Already implemented in the project — the single shared WebGL renderer uses scissor tests to render each card's viewport. No additional renderers needed.
- **IntersectionObserver**: Already implemented — off-screen robots skip rendering.
- **Animation is cheap**: Only group rotations and position offsets. No geometry changes, no texture updates.

### Estimated mesh counts per robot:
| Style | Meshes | Est. Triangles |
|-------|--------|----------------|
| Cute | ~14 | ~2,500 |
| Mech | ~18 | ~1,800 (all boxes) |
| Retro | ~22 | ~1,600 (all boxes) |
| Drone | ~20 | ~3,200 |

---

## 8. Variation / Randomization

For extra variety when showing multiple robots of the same style, you can apply minor randomization:

```js
// Color hue shift — rotate the palette d offset slightly
const hueShift = (Math.random() - 0.5) * 0.1;
mat.uniforms.uPalD.value.x += hueShift;
mat.uniforms.uPalD.value.y += hueShift;

// Size variation +-10%
const scale = 0.9 + Math.random() * 0.2;
robot.root.scale.setScalar(scale);

// Animation phase offset — prevents all robots from bobbing in sync
const phaseOffset = Math.random() * Math.PI * 2;
// Use (t + phaseOffset) instead of t in animation function
```

---

## Summary

| Aspect | Approach |
|--------|----------|
| Body construction | Built-in geometries (Box, Sphere, Capsule, Cylinder) grouped with THREE.Group |
| Joint articulation | Nested THREE.Group at pivot points, animate via group.rotation |
| Material (body) | Project's existing `createEditorMaterial(palette)` ShaderMaterial |
| Material (accents) | `MeshBasicMaterial` for self-illuminated eyes/lights |
| Animation | Idle loops: float, head turn, breathing, eye glow, style-specific limb motion |
| Card integration | New `type: '3d-robot'` in cards array, handled in `createMiniScene()` |
| Performance | ~14-22 meshes per robot, ~2-3k triangles, single shared renderer with scissor |
| Scale | Robot fits in 2x2x2 cube, camera at z=3.5, FOV 35 |
