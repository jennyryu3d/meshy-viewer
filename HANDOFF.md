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

## v15 — 조인트 회전 기즈모(매니퓰레이터)
- `TransformControls`(three examples) 로드 → 선택 본에 **회전 기즈모** 표시. `initGizmo()`에서
  생성(rotate 모드, local 공간, size 0.8), `updateGizmo()`가 선택 본에 attach/detach.
  "조인트 기즈모 표시" 체크박스(`gizmoChk`)로 on/off.
- 드래그 처리: `dragging-changed`→OrbitControls 비활성+일시정지, `objectChange`→
  `editOverride={bone, q:bone.quaternion}` 갱신 + 오일러 필드 동기화. 기존 editOverride/animate
  덮어쓰기 메커니즘으로 프리뷰 유지, "키 저장"은 editOverride.q(정확값) 사용.
- 주의: `gizmo`도 상단 전역. CDN에서 TransformControls.js 추가 로드(없으면 initGizmo no-op).

## v14 — 재생 컨트롤 + 조인트 편집 + 그래프
- **재생 컨트롤**: 일시정지/재생(`togglePlay`, action.paused 사용), 한 프레임씩(`stepFrame±1`),
  스크럽 슬라이더, frame 입력. 키보드: space=재생/정지, `,`/`.`=프레임. 시킹은 `seekFrame(f)`
  → ensureSolo(다른 액션 stop) + active.time=f/fps + mixer.update(0). fps는 클립 최대 트랙 키수로 추정.
- **조인트 편집**: 본 드롭다운(`boneSel`, 캐릭터 스켈레톤), 오일러 XYZ(deg) 입력. 편집 시
  `editOverride={bone,q}`로 매 프레임 animate에서 로컬 회전 덮어써 프리뷰(일시정지 자동). 
  "이 프레임에 키 저장"=`saveKey`: **base 클립**의 해당 본 quaternion 트랙에서 현재 시간 최근접
  키 index에 기록(트랙 없으면 생성) → playCache 비우고 rebuildActions → 같은 프레임 재시킹.
  base에 저장하므로 drift/footlock 토글에도 유지. 되돌리기=프리뷰 폐기 후 재시킹.
- **그래프**(`#graph` canvas): 선택 본 로컬 회전 오일러 X/Y/Z(빨강/초록/파랑) 전체 구간 + 현재
  프레임 세로선(주황). graphCache는 본/저장 시 갱신.
- 주의: editor 상태 변수(editSkel/editOverride/graphCache 등)는 **반드시 상단 전역**에 선언할 것
  — animate()가 init()에서 동기 호출되며 editOverride를 읽으므로 TDZ 에러 방지.

## v13 — 발 고정 기본 OFF(충실 모드) + 타임코드 표시
사용자 피드백: IK 처리(발 고정/무릎 pole)가 전체 자연스러움을 떨어뜨림 → 발이 미끄러져도
DeepMotion 원본 그대로가 더 나음.
- **발 고정 체크박스(`flChk`) 추가, 기본 OFF.** OFF면 `footLockClip` 미실행 → v9 충실 모드
  (rest-pose 보정 리타겟 + hips 접지 + 좌측 다리 매칭만, IK 없음). 켜면 v10~12의 발 고정/무릎
  pole IK 적용. `buildPlayClip` 캐시키에 fl 상태 포함, 토글 시 `rebuildActions`.
- `detectSourceContacts`는 여전히 리타겟 시 계산(토글 켤 때 재리타겟 없이 쓰려고). off여도 비용
  약간 있음(소스 1패스). 싫으면 lazy화 가능하나 소스 GLB는 재생 시점에 없으므로 리타겟 때 해야 함.
- **타임코드 HUD(`#timecode`)**: 재생 중 `클립명 · t 현재/총 s · f 프레임/총`. fps는 클립 최대
  트랙 키수/duration으로 추정. animate()에서 `updateTimecode()`. 우상단(모바일은 하단 중앙).

## v12 — 무릎 역관절 제거 (knee pole IK)
v11에서도 무릎 역관절이 미세하게 남음. 원인: 회전 리타게팅(C 켤레)이 무릎 hinge 축을 완벽히
보존 못해 일부 프레임 무릎이 반대로 굽음(타깃 무릎 vs 소스 무릎 일치도 0.92, 가끔 반대).
- 해결: 2본 IK에 **명시적 pole** 추가 → 무릎이 pole 방향으로만 굽음. pole은 **소스 무릎의
  실제 굽힘 방향**(hip→ankle 선에서 무릎 오프셋, 월드)을 사용. 소스는 해부학적으로 유효하고
  타깃과 같은 방향을 보므로, 그 방향이 곧 타깃 무릎이 굽어야 할 방향. `detectSourceContacts`가
  접촉+무릎 pole을 같이 반환({contacts, poles}). 모든 프레임에 IK 적용(비잠금 프레임은 target=
  현재 ankle이라 발은 그대로, 무릎 방향만 교정).
- 부작용 보정: 매 프레임 IK 재계산이 무릎 jitter를 키워서(평균 7→12mm) `smoothQuats`(slerp 1패스,
  부호 정렬)로 완화 → FK 수준 복귀.
- 안전장치: pole 없을 때 몸통-forward(양 힙 cross up)×지배 방향 sign으로 fallback. reach 0.985 클램프.
- **측정**: 타깃 무릎 vs 소스 무릎 일치도 0.92→**0.99**, 역관절 프레임 0. jitter FK 수준(무릎 8.0
  vs FK 7.1, 발목은 FK보다 낮음). plant skate 2.38→1.89, 걸음 범위 FK 유지.
- 주의: 무릎 일치도 측정은 **타깃 무릎 perp vs 소스 무릎 perp** 비교가 정답(발-forward 기준
  지표는 유효 소스도 오탐하므로 쓰지 말 것). diag3.js 참고.

## v11 — 발 고정을 소스 접촉 기반으로 (v10 과잉 잠금/역관절 수정)
v10은 타깃(리타겟된) 발에서 접촉을 감지하고 공격적으로 잠가서 **스윙(걸음)까지 묶임** → 발이
한 점에 붙박이고, 몸이 그 위로 움직이며 **무릎 역관절** 발생.
- 핵심 교훈: 리타겟된 타깃 발 모션은 노이즈가 커서 접촉 판정이 부정확. **소스(딥모션) 발은
  접지가 깨끗**(좌 16/우 15개의 명확한 plant 구간). → `detectSourceContacts()`로 **소스에서
  발 접촉 타이밍을 추출**해 클립에 저장(`out.footContacts`), `footLockClip`은 그 타이밍에만 잠금.
  즉 소스가 디딜 때만 디디고, 소스가 뗄 때 같이 뗌 → 걸음 안 묶임.
- 안전장치: 2본 IK reach를 0.985로 클램프(완전 폄 직전 → 역관절 특이점 회피). 짧은 ramp(0.006)
  로 plant 구간을 단단히 고정하되 release가 실제 liftoff에 떨어져 pop 없음. cloneClip이
  `footContacts`/`retargeted` 메타 보존.
- **측정**: plant 중 발 skate FK 2.38→1.63(32%↓), 걸음 범위 FK의 98~100%(안 묶임), 역관절 0,
  최대 저크 = 소스와 동일(pop 없음). hips 불변(충실).
- **솔직한 한계**: 이 클립은 댄스라 발이 지면 근처에서 계속 움직임(소스 자체가 shuffle).
  깨끗한 plant 구간만 잠그므로 잔여 미끄러짐은 남음(소스 모션 보존과의 본질적 trade-off).
  더 강하게 잠그면 v10처럼 걸음이 묶임. 임계값은 footLockClip/detectSourceContacts 상단.

## v10 — (대체됨, v11 참조) 2본 IK 발 고정 첫 시도
v9는 딥모션 키에 충실하지만 접지 중인 발이 미끄러지는 게 남음. v10에서 **발 고정(foot-lock)
+ 2본 IK** 추가:
- 핵심: **루트(hips)는 손대지 않고**(딥모션 그대로=떨림 無) **다리(엉덩이+무릎)만 IK로 굽혀**
  접지 중인 발을 고정 월드 지점에 핀. → 미끄러짐 제거 + 루트 떨림 없음.
- 접촉 감지: 발 높이 < 20퍼센타일+0.20·다리길이 AND 수평속도 < 2.6·다리길이/s. 짧은 갭은
  병합(0.10·nf), 짧은 구간 제거(0.012·nf). 각 구간은 median 발 위치에 잠그고, 발의 최저점이
  바닥에 닿도록 발목 Y 보정. 진입/이탈은 **smoothstep**로 블렌딩(전환 자연스럽게).
- `footLockClip()`은 `buildPlayClip()`에서 drift strip **후** 실행(루트 상태 반영). retargeted
  클립에만 적용(Meshy 클립은 그대로). 결과는 (name|drift)로 캐시. ~27ms.
- 쿼터니언 부호 연속성 보정(writeQuatTrack)으로 slerp pop 방지. 2본 IK는 aim→무릎 굽힘(FK
  굽힘 평면 유지)→재aim.
- **측정**: 접지 중 발 skate 11.4→0.66(94%↓). 프레임 평균 저크 21.9→8.5mm(소스보다 부드러움!
  발을 잡아주니 떨림도 같이 줄어듦). hips 저크 2.99(불변=충실). 최대 저크 480 vs 소스 384
  (한 프레임만 약간 큼, 댄스 자체가 격해 묻힘).
- 한계: groundClip(상수 오프셋)+footLock 조합. 접촉이 거의 없는 모션(공중기 등)은 잠글 게
  없음. 임계값은 딥모션 댄스 클립 기준 튜닝(footLockClip 상단 상수).

## v8 — 왼쪽 다리 뻣뻣함 수정
`canonBone()`가 `LeftLeg`(core 'leg')의 앞 'l'을 무조건 떼서 'eg'→null로 만들어 **왼쪽
정강이 매칭 실패**(bind 고정 = 뻣뻣). 오른쪽은 'leg'가 'r'로 시작 안 해 멀쩡. → core를
**있는 그대로 먼저** 확인하고 안 맞을 때만 앞 글자 strip(DeepMotion 'l_leg'→'lleg'→'leg'
대응). 매칭 21→22/24. LeftLeg 회전 진폭 0°→127°.

## v9 — 프레임마다 불안정(키 출렁임) 수정
증상: 리타게팅 모션이 소스(DeepMotion)보다 훨씬 불안정, 프레임마다 캐릭터 키/높이가 달라 보임.
원인: **`groundClip`이 매 프레임 발 높이에 맞춰 hips를 올림**. 소스 루트(hips)는 캡처 품질
탓에 이미 출렁이는데, 거기에 per-frame 보정이 더해져 hips Y 떨림이 소스의 ~1.5배로 증폭됨.
→ `groundClip`을 **상수 오프셋 1개**(전 구간 발 최저높이 20퍼센타일을 바닥으로)만 적용하도록
변경. 소스 수직 모션을 그대로 보존(=DeepMotion 키에 충실), hips Y 떨림이 소스와 **동일**해짐.
   - 측정: hips Y jitter ratio 0.019→**0.0126(=소스 0.0126)**. 본 길이 일정(stretch 없음),
     쿼터니언 단위(노름편차 2e-7). Timeline_2는 보행이 아니라 **킥/점프 있는 댄스** 모션이라
     발이 자주 뜨는 건 소스 그대로(정상).
   - 트레이드오프: 발을 매 프레임 바닥에 붙이려면(per-frame) hips가 떨리고, 충실하려면
     상수 오프셋이라 다리 리타게팅 오차만큼 일부 프레임 발이 뜨거나 살짝 묻음. 사용자 요청
     (충실 우선)에 따라 상수 오프셋 선택. 발 접지를 더 원하면 per-frame을 **저역통과 필터**로
     부드럽게 하는 방향이 다음 후보.
v6에서 Timeline_2 재생 시 캐릭터가 그리드 아래로 가라앉는 문제 수정. 원인 2가지:
1. **degenerate bind pose.** Meshy 리그의 `inverseBindMatrices`(=`skeleton.pose()` 결과)는
   다리가 거의 0으로 접힌 비정상 bind라, pose() 후 월드 거리/스케일이 붕괴됨(다리 0.0087 world).
   실제 재생은 authored 스케일(다리 ~0.81 world). → leg length는 **로컬 오프셋 합**(`bone.position`)
   으로 측정해야 안전. hips 높이 baseline도 bind hips(0.94)가 아니라 **소스의 standing hip
   월드 높이**(소스 bind는 정상 standing)로 스케일.
3. **foot grounding 추가(`groundClip`).** 리타게팅 클립을 캐릭터에 임시 mixer로 샘플링해,
   매 키프레임 hips Y를 올려 **최저 발이 y=0**에 닿게 함. pose-driven이라 자연스러운 수직
   bob도 보존됨. x/z(좌우 이동)는 소스에서 스케일해 유지. **반드시 authored pose/scale로
   복원한 뒤** grounding 실행(안 그러면 degenerate 스케일로 샘플링돼 무의미).
   - 검증: hips 월드 Y 0.80~0.96, 최저 발 0~0.07(그리드 위), drift on=제자리+bob, off=이동.

### 주의 (다음 사람용)
- Meshy `character.glb`의 Armature 스케일이 **0.01**. bind pose(`skeleton.pose()`)는 실제
  standing 포즈와 다른 degenerate 상태 — 월드 거리 측정 신뢰 불가, 로컬 오프셋 사용할 것.
- 발/발가락(LeftFoot/ToeBase)은 rest 축 차이 + 빠른 모션으로 순간 회전 편차가 큼(시각적으론 OK).
- `groundClip`은 매 프레임 최저 발을 바닥에 붙이므로 **점프(양발이 뜨는 모션)** 는 눌릴 수 있음.
  현재 자산(보행)엔 문제없음. 점프 클립 필요 시 grounding을 상수 오프셋 방식으로 바꿀 것.

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
