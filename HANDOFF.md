# Meshy 웹 뷰어 — 리타게팅 작업 핸드오프

## 지금까지 (작동하는 것)
- `index.html` (= meshy-viewer, 현재 **v5**): Three.js r128 기반 GLB 캐릭터 뷰어.
- 자동 로드(`character.glb` + `walking/running/jumping.glb`), 파일 pick/drop, URL 로드.
- 카메라 자동 프레이밍(스켈레톤 본 위치 기준), A=전체맞춤 / F=원점.
- 모바일: 패널 접기 + 하단 클립바.
- **Meshy 자체 클립**(같은 골격)은 완벽히 재생됨. 멀티 클립 버튼 전환 OK.
- 외부 클립(DeepMotion) 추가 시 **본 이름 자동 매칭은 성공** → `retargeted 21/24 bones`.

## 해결됨 (v6) ✅
1. **rest-pose 보정 리타게팅 구현 완료.** `SkeletonUtils.retargetClip`(이름만 변경)을
   버리고 `retargetToCharacter()`를 직접 구현. 본별로 두 골격의 **rest world 회전**으로
   소스 로컬 회전을 타깃 로컬 공간으로 변환:
   - `qsΔ = qsRest⁻¹ · qs` (소스 rest로부터의 모션 델타, 소스 본 프레임)
   - `C = Wt⁻¹ · Ws` (본별 상수: 소스 rest 프레임 → 타깃 프레임. Ws/Wt = rest world 회전)
   - `qt = qtRest · C · qsΔ · C⁻¹` (같은 월드 모션을 타깃 로컬 프레임에 재표현)
   - rest(qs=qsRest)에서 qt=qtRest로 collapse → 변화 없음(올바름).
   - **검증:** DeepMotion `Timeline_2`가 character.glb에서 자연스러운 보행으로 재생됨.
     소스 대비 월드 회전 모션 차이 평균 ~2–6° (T-pose면 100°+). Playwright 스크린샷 확인.
2. **drift strip 실시간 토글 완료.** 원본(=base) 클립을 `clipBases`에 보관, 재생용 클립은
   `buildPlayClip()`이 체크박스 상태에 따라 base 그대로 / strip한 clone을 반환.
   체크박스 `onchange` → `rebuildActions()`가 새 mixer로 액션 재생성(현재 클립·재생 위치 유지).
3. **hips 위치 스케일 보정 완료.** position 트랙은 **로컬** 공간이므로 **로컬** rest hip
   높이 비율(`ptRest.y/psRest.y`)로 스케일 (월드 높이는 armature 스케일이 섞여 틀림 — Meshy
   Armature는 ×0.01). 부모 world 회전으로 이동 방향 재표현. drift off 시 좌우 이동 살아남.

### 주의 (다음 사람용)
- Meshy `character.glb`의 Armature 스케일이 **0.01** → 본은 로컬 ~0.94, 월드 hip 높이 ~0.0094m.
  카메라 fitAll이 알아서 프레이밍하므로 보기엔 정상. 위치 계산 시 로컬/월드 단위 혼동 주의.
- 발/발가락(LeftFoot/ToeBase)은 rest 축 차이 + 빠른 모션으로 순간 회전 편차가 큼(시각적으론 OK).

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
