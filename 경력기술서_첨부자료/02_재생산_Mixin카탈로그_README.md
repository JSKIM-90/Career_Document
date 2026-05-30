# Mixin 가이드

## Mixin을 만드는 기준

**"이 Mixin의 기능이 무엇인가."**

기능이 기존 Mixin과 다르면 새 Mixin이다. 같으면 기존 Mixin을 사용한다.

기능 = 보편화된 목적(행위)을 특수한 상황과 결합한 단위. **목적이 같아도 수단이 다르면 다른 기능**이다. 예를 들어 FieldRenderMixin과 MeshStateMixin은 "데이터를 보여준다"는 같은 목적이지만, 수단이 다르므로(DOM textContent vs 3D material 색상) 별도 Mixin이다.

목적이 같고 수단도 비슷해 보이는 경우, 다른 부분이 설정으로 분리 가능하면 같은 기능(하나의 Mixin)이고, 불가능하면 다른 기능(별도 Mixin)이다.

> 기능의 정의에 대한 상세: [COMPONENT_SYSTEM_DESIGN.md — 기능의 정의](01_아키텍처_컴포넌트시스템설계.md#기능의-정의)

---

## 현재 Mixin 목록

| Mixin | 목적 |
|-------|------|
| FieldRenderMixin | 기존 요소에 데이터를 표시한다 |
| ListRenderMixin | 항목을 반복 생성하여 표시한다 (`itemKey` 옵션 시 개별 항목 상태 변경/조회 가능) |
| TreeRenderMixin | 계층적 데이터를 트리 구조로 렌더링한다 (재귀적 중첩 + 확장/축소) |
| EChartsMixin | ECharts 인스턴스를 생성하고 관리한다 |
| TabulatorMixin | Tabulator 인스턴스를 생성하고 관리한다 |
| HeatmapJsMixin | heatmap.js 기반 열 분포를 시각화하고 관리한다 |
| MeshStateMixin | 3D 메시의 시각 상태를 데이터에 따라 변경한다 |
| MeshHighlightMixin | 선택한 대상을 시각적으로 강조한다 |
| MeshVisibilityMixin | 3D 장면의 특정 부분을 선택적으로 보여준다 |
| CameraFocusMixin | 보는 위치를 전환한다 |
| AnimationMixin | 3D 모델의 동작을 재현한다 (GLTF AnimationClip 재생/정지/속도 제어) |
| ClippingPlaneMixin | 3D 모델의 내부를 보여준다 (ClippingPlane으로 절단면 표시) |
| ShadowPopupMixin | Shadow DOM으로 팝업을 생성하고 표시/숨김을 관리한다 |
| 3DShadowPopupMixin | 3D 컴포넌트에서 Shadow DOM으로 팝업을 생성하고 표시/숨김을 관리한다 |
| ComponentMixin | 뷰어 전용 라이프사이클 훅(`_onViewerReady`/`_onViewerDestroy`) 규약과 3D ModelLoader 리소스 로딩(`applyModelLoaderMixin`)을 제공한다 |

---

## Mixin 공통 원칙

- **각 Mixin이 받는 옵션이 그 Mixin의 인터페이스다.** 옵션은 Mixin별 문서를 참고한다.
- **데이터 변환은 Mixin의 관심사가 아니다.** Mixin은 이미 가공된 데이터만 받는다.
- **Mixin은 자기 것만 관리한다.** destroy에서 자기가 만든 것만 정리한다.

---

## Mixin별 상세 문서

| Mixin | 문서 |
|-------|------|
| FieldRenderMixin | FieldRenderMixin.md |
| ListRenderMixin | ListRenderMixin.md |
| TreeRenderMixin | TreeRenderMixin.md |
| EChartsMixin | EChartsMixin.md |
| TabulatorMixin | TabulatorMixin.md |
| HeatmapJsMixin | HeatmapJsMixin.md |
| MeshStateMixin | MeshStateMixin.md |
| MeshHighlightMixin | MeshHighlightMixin.md |
| MeshVisibilityMixin | MeshVisibilityMixin.md |
| CameraFocusMixin | CameraFocusMixin.md |
| AnimationMixin | AnimationMixin.md |
| ClippingPlaneMixin | ClippingPlaneMixin.md |
| ShadowPopupMixin | ShadowPopupMixin.md |
| 3DShadowPopupMixin | 3DShadowPopupMixin.md |
