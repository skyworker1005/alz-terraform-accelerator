
이 리포지토리는 **Azure Landing Zones (ALZ) Terraform Accelerator**의 스타터 템플릿입니다. 분석 결과를 요약합니다.

---

## 1. 프로젝트 개요

- **목적**: Azure Landing Zones를 Terraform(IaC)으로 배포하기 위한 스타터 모듈 제공  
- **공식 문서**: [aka.ms/alz/acc](https://aka.ms/alz/acc)

---

## 2. 디렉터리 구조

| 경로 | 역할 |
|------|------|
| `templates/platform_landing_zone/` | **핵심** – 플랫폼 랜딩존 배포 템플릿 |
| `templates/platform_landing_zone/modules/config-templating/` | 설정 템플릿(치환·리전·구독 ID 등) 처리 |
| `templates/platform_landing_zone/lib/` | ALZ 아키텍처/아키타입 정의(YAML) |
| `templates/test/` | E2E 테스트용 간단한 ALZ 배포 |
| `templates/empty/` | 빈 템플릿(README 등만 있음) |

---

## 3. 사용 기술 스택

- **Terraform**: `~> 1.12`
- **Provider**:
  - **Azure/alz** (0.20.2): ALZ 라이브러리(YAML 기반 정의 로드)
  - **hashicorp/azurerm** (~> 4.0): Azure 리소스
  - **Azure/azapi** (~> 2.0): ARM/미지원 리소스
  - **hashicorp/local** (~> 2.5): 로컬 연산

- **ALZ Provider 설정**:  
  `path.root/lib`를 커스텀 라이브러리 경로로 사용해, 로컬 YAML로 아키텍처/아키타입을 덮어씀.

---

## 4. 아키텍처 구성

### 4.1 구독 분리

- **management**: 관리(모니터링, 로그 등)
- **connectivity**: 네트워크(허브/스포크, Virtual WAN 등)
- **identity**: ID(선택)
- **security**: 보안(선택)

변수 `subscription_ids`로 지정하며, 기존 `subscription_id_*` 변수는 deprecated.

### 4.2 Management Group 계층

`lib/architecture_definitions/alz_custom.alz_architecture_definition.yaml` 기준:

- **alz** (루트) → **platform** / **landingzones** / **sandbox** / **decommissioned**
  - **platform** 아래: **security**, **management**, **connectivity**, **identity**
  - **landingzones** 아래: **corp**, **online**

각 노드는 `lib/archetype_definitions/*.alz_archetype_override.yaml`로 정책/역할 등 세부 정의.

### 4.3 Connectivity 옵션 (3가지)

| 값 | 설명 |
|----|------|
| `hub_and_spoke_vnet` | Hub-and-Spoke VNet (기본값) |
| `virtual_wan` | Virtual WAN |
| `none` | 네트워크/커넥티비티 리소스 없음(관리만) |

`connectivity_type`으로 선택하며, `locals.tf`에서 `connectivity_*_enabled`로 분기.

---

## 5. 주요 모듈 흐름

1. **config-templating** (`main.config.tf`)
   - 입력: `starter_locations`, 구독 ID, `custom_replacements`, connectivity/management 설정 등
   - `templatestring`으로 플레이스홀더 치환 후 JSON 디코드
   - 출력: `management_group_settings`, `management_resource_settings`, `connectivity_resource_groups`, `hub_and_spoke_networks_settings`, `virtual_wan_settings` 등
   - **항상 먼저 실행**되고, 다른 모듈은 이 출력에 의존

2. **resource_groups** (`main.resource.groups.tf`)
   - Connectivity 구독에 리소스 그룹 생성 (`avm-res-resources-resourcegroup`)
   - `module.config.outputs.connectivity_resource_groups` 기반, `for_each`로 RG 다수 생성

3. **management_groups** (`main.management.groups.tf`)
   - `management_groups_enabled == true`일 때만
   - `Azure/avm-ptn-alz`로 MG 계층, 정책, 역할 할당 배포
   - 정책 할당은 `management_resources`, `hub_and_spoke_vnet`, `virtual_wan` 등에 대한 의존성(`policy_assignments_dependencies`)으로 순서 보장

4. **management_resources** (`main.management.resources.tf`)
   - `management_resources_enabled == true`일 때만
   - Log Analytics, 데이터 수집 규칙, Sentinel, 관리용 관리 ID 등 (`Azure/avm-ptn-alz-management`)

5. **virtual_wan** / **hub_and_spoke_vnet**
   - `connectivity_type`에 따라 둘 중 하나만 배포
   - Virtual WAN: `avm-ptn-alz-connectivity-virtual-wan`
   - Hub-and-Spoke: `avm-ptn-alz-connectivity-hub-and-spoke-vnet`
   - 둘 다 connectivity 구독, `azurerm.connectivity` / `azapi.connectivity` 사용

---

## 6. Config-Templating 동작

- **역할**: 사용자/예제에서 넘긴 설정을 “이름·리전·구독 ID” 등으로 치환해, ALZ 모듈이 기대하는 형태로 만듦.
- **치환 단계** (순서):
  1. `built_in_replacements`: `starter_locations`, `starter_locations_short`, `root_parent_management_group_id`, 구독 ID들
  2. `custom_replacements.names` → `custom_name_replacements`
  3. `custom_replacements.resource_group_identifiers` → `custom_resource_group_replacements`
  4. `custom_replacements.resource_identifiers` → `final_replacements`
- `regions` 모듈로 리전 메타데이터(geo_code, short_name 등) 조회 후, `templatestring`으로 JSON 문자열 치환하고 다시 `jsondecode`해 구조화된 `outputs` 생성.

---

## 7. 출력(Outputs)

- **Connectivity**: DNS 서버 IP, VNet/방화벽/베스천 리소스 ID·이름·IP, 라우트 테이블 등 (Hub-and-Spoke / Virtual WAN별로 분리)
- **templated_inputs**: `module.config.outputs` 전체 (다른 레이어나 스크립트에서 재사용 가능)

---

## 8. 배포 제어 플래그

- `management_groups_enabled`: MG 계층 배포 여부
- `management_resources_enabled`: 관리 리소스(LAW, Sentinel 등) 배포 여부
- `connectivity_type`: `"none"` | `"hub_and_spoke_vnet"` | `"virtual_wan"`
- `enable_telemetry`: 원격 원격 분석 전송 여부

---

## 9. 정리

- **config-templating**이 모든 설정의 중심이고, 여기서 나온 출력으로 **Resource Groups → Management Groups / Management Resources / Connectivity** 순으로 배포됩니다.
- **ALZ Provider**의 `library_overwrite_enabled`와 `path.root/lib`로 로컬 YAML이 적용되므로, `lib/` 아래 아키텍처·아키타입 YAML을 수정하면 정책/계층 구조를 커스터마이즈할 수 있습니다.
- 실제 사용 시에는 `templates/platform_landing_zone/examples/` 아래의 `*.tfvars`(single-region, multi-region, management-only, migration 등)를 참고해 변수만 채워서 사용하는 구조입니다.

특정 파일(예: `main.config.tf`, `locals.tf`, config-templating 내부)이나 배포 시나리오(예: Virtual WAN만 쓰는 경우)를 더 깊게 보고 싶으면 그 부분을 지정해 주시면 됩니다.