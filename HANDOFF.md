# Meshy 웹 뷰어 — 리타게팅 작업 핸드오프

## 지금까지 (작동하는 것)
- `index.html` (= meshy-viewer, 현재 **v5**): Three.js r128 기반 GLB 캐릭터 뷰어.
- 자동 로드(`character.glb` + `walking/running/jumping.glb`), 파일 pick/drop, URL 로드.
- 카메라 자동 프레이밍(스켈레톤 본 위치 기준), A=전체맞춤 / F=원점.
- 모바일: 패널 접기 + 하단 클립바.
- **Meshy 자체 클립**(같은 골격)은 완벽히 재생됨. 멀티 클립 버튼 전환 OK.
- 외부 클립(DeepMotion) 추가 시 **본 이름 자동 매칭은 성공** → `retargeted 21/24 bones`.

## 막힌 것 (핵심 버그)
1. **리타게팅된 클립이 깨진다.** `Timeline_2`(DeepMotion) 버튼 → 캐릭터가 T-pose로 정지하거나 누운 포즈 등 엉뚱하게 감. 본은 매칭됐지만 회전이 잘못 들어감.
   - 원인: `THREE.SkeletonUtils.retargetClip`이 **두 골격의 rest(bind) 포즈·본 로컬 축 차이를 보정하지 못함.** 이름만 맞추면 회전 데이터가 어긋남.
2. **"제자리 고정"(drift strip) 토글이 재생 중 반영 안 됨.** 현재 클립 *추가 시점*에만 `stripRootDrift()`가 적용됨. 실시간 토글로 바꿔야 함.

## 다음 작업 (TODO)
1. **rest-pose 보정 리타게팅 재구현.** retargetClip 대신, 본별로
   `targetBindLocal⁻¹ · (sourceParentWorld⁻¹ · sourceBoneWorld) · ...` 형태의
   bind-pose delta를 계산해 소스 회전을 타깃 로컬 공간으로 변환. 즉
   `q_target_local = Bt⁻¹ · Bs · q_source_local · Bs⁻¹ · Bt` 류의 retarget.
   (참고: three의 retargetClip 소스, 또는 mixamo→VRM 리타게팅 구현들.)
   - 검증: walk/idle 같은 명확한 모션이 팔 꼬임 없이 자연스럽게 나와야 함.
2. **drift strip을 실시간 토글로.** 원본 트랙을 보관하고, 체크박스 변경 시
   활성 액션을 재생성(원본 ↔ strip 버전 스왑).
3. (선택) hips 위치 스케일 보정 — 소스/타깃 키 비율로 hip translation 스케일.

## 검증된 사실 — 본 이름 맵 (이게 핵심 자산)
DeepMotion(소스, `*_JNT`) → Meshy(타깃):
```
hips_JNT      → Hips
spine_JNT     → Spine
spine1_JNT    → Spine01
spine2_JNT    → Spine02
neck_JNT      → neck
head_JNT      → Head
l_shoulder_JNT→ LeftShoulder     r_shoulder_JNT→ RightShoulder
l_arm_JNT     → LeftArm          r_arm_JNT     → RightArm
l_forearm_JNT → LeftForeArm      r_forearm_JNT → RightForeArm
l_hand_JNT    → LeftHand         r_hand_JNT    → RightHand
l_upleg_JNT   → LeftUpLeg        r_upleg_JNT   → RightUpLeg
l_leg_JNT     → LeftLeg          r_leg_JNT     → RightLeg
l_foot_JNT    → LeftFoot         r_foot_JNT    → RightFoot
l_toebase_JNT → LeftToeBase      r_toebase_JNT → RightToeBase
```
- Meshy엔 손가락 본 없음(DeepMotion 손가락 트랙은 버림). Meshy의 `head_end`,`headfront`는 소스 대응 없음.
- 코드의 `canonBone()` 정규화 함수가 위 매칭을 자동 생성함(Mixamo `mixamorig:` 도 대응). 이 함수는 유지해도 됨 — 문제는 매칭이 아니라 **회전 변환**.

## 환경/제약
- Three.js r128 (jsdelivr UMD: three.min.js + examples/js/{controls/OrbitControls, loaders/GLTFLoader, utils/SkeletonUtils}).
- 배포: GitHub Pages, `index.html` + GLB들 레포 루트. URL 자동 fetch(같은 도메인이라 CORS OK).
- 모바일 캐시 끈질김 → 헤더에 버전 배지 + ⟳(hardReload, `?cb=` 쿼리) 있음. 수정 시 `APP_VERSION` 올릴 것.
- 챗 미리보기는 외부 fetch 차단 → 테스트는 GitHub Pages/로컬에서.

## 테스트 자산
- Meshy 캐릭터: `character.glb` (24본, `Hips/Spine/LeftArm…`), 임베디드 idle 클립.
- 외부 목캡: DeepMotion GLB (52본 `*_JNT`, 클립 `Timeline_2`, hips에 translation=좌우 이동).
- 검증 기준: Timeline_2가 캐릭터에서 **자연스러운 사람 동작**으로 재생 + (drift off 시) 좌우 이동 살아남.
