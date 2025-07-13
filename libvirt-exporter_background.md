# OpenStack Nova 컴퓨트 노드의 메트릭 측정을 위한 LIBVIRT-EXPORTER의 배경

## 개요

OpenStack 기반의 가상화 인프라에서 Nova 컴퓨트 서비스는 VM(인스턴스)의 생성, 삭제, 상태 관리 등의 주요 역할을 담당합니다. Nova는 직접 하이퍼바이저를 제어하지 않고, 하위 계층의 libvirt를 통해 KVM/QEMU와 같은 하이퍼바이저와 인터페이스합니다.

즉, Nova는 VM 관리의 오케스트레이션 계층이라면, libvirt는 실제 VM의 라이프사이클과 리소스 상태를 제어하는 실행 계층입니다.

## 모니터링의 필요성

### 1. OpenStack API 기반 모니터링의 한계

OpenStack의 각 서비스는 REST API를 통해 메트릭을 제공하지만, 다음과 같은 한계가 있습니다:

- **확장성 문제**: 대규모 클라우드에서는 API 요청이 증가하면서 응답 시간이 지연됩니다
- **API 오버헤드**: 여러 서비스를 거쳐 데이터를 수집하는 과정에서 성능 저하가 발생합니다
- **실시간성 부족**: API 응답 지연으로 인한 실시간 모니터링의 어려움
- **세밀한 메트릭 부족**: VM의 실제 하드웨어 리소스 사용량에 대한 상세한 정보 제공 제한

### 2. libvirt를 통한 직접 모니터링의 필요성

OpenStack 환경에서 실제 VM의 리소스 상태를 정확히 파악하기 위해서는 libvirt를 통한 직접적인 메트릭 수집이 필요합니다:

- **실시간 데이터**: libvirt는 하이퍼바이저에서 직접 데이터를 수집하므로 실시간 모니터링이 가능
- **상세한 리소스 정보**: CPU, 메모리, 디스크 I/O, 네트워크 등의 세밀한 성능 메트릭 제공
- **하드웨어 수준 정보**: 하이퍼바이저 레벨에서의 정확한 리소스 사용량 측정

## LIBVIRT-EXPORTER의 역할

### 1. 기본 개념

LIBVIRT-EXPORTER는 Prometheus 메트릭 수집기로, libvirt 데몬에 직접 연결하여 가상 머신의 성능 메트릭을 수집하고 Prometheus 형식으로 노출합니다.

### 2. 주요 기능

#### 메트릭 수집 범위
- **도메인(VM) 정보**: 상태, CPU 시간, 메모리 사용량, 가상 CPU 수
- **블록 디바이스 통계**: 읽기/쓰기 바이트, 요청 수, 응답 시간
- **네트워크 인터페이스 통계**: 송수신 바이트, 패킷 수, 오류 및 드롭 수
- **메모리 상세 정보**: 실제 사용량, 캐시, 버퍼 등

#### OpenStack 특화 기능
- **Nova 메타데이터 지원**: 인스턴스 이름, 프로젝트 정보, 사용자 정보, 플레이버 정보
- **프로젝트별 필터링**: 특정 프로젝트의 인스턴스만 모니터링 가능
- **UUID 매핑**: libvirt 도메인 이름과 OpenStack 인스턴스 UUID 연결

### 3. 수집 가능한 메트릭 예시

```
# CPU 관련 메트릭
libvirt_domain_info_cpu_time_seconds_total{domain="instance-00000337"} 949422.12
libvirt_domain_info_virtual_cpus{domain="instance-00000337"} 2
libvirt_domain_vcpu_time_seconds_total{domain="instance-00000337",vcpu="0"} 315190.41

# 메모리 관련 메트릭
libvirt_domain_info_maximum_memory_bytes{domain="instance-00000337"} 8589934592
libvirt_domain_info_memory_usage_bytes{domain="instance-00000337"} 8589934592
libvirt_domain_memory_stats_available_bytes{domain="instance-00000337"} 8363945984

# 디스크 I/O 메트릭
libvirt_domain_block_stats_read_bytes_total{domain="instance-00000337",target_device="sda"} 177040343040
libvirt_domain_block_stats_write_bytes_total{domain="instance-00000337",target_device="sda"} 921412177920
libvirt_domain_block_stats_read_requests_total{domain="instance-00000337",target_device="sda"} 19613982

# 네트워크 I/O 메트릭
libvirt_domain_interface_stats_receive_bytes_total{domain="instance-00000337",target_device="tapa7e2fe95-a7"} 7918228100
libvirt_domain_interface_stats_transmit_bytes_total{domain="instance-00000337",target_device="tapa7e2fe95-a7"} 1819996331
```

## 기술적 구현

### 1. libvirt API 활용

LIBVIRT-EXPORTER는 libvirt의 공식 Go 바인딩을 사용하여 다음 API를 활용합니다:

- **GetAllDomainStats()**: 모든 도메인의 통계를 효율적으로 수집
- **DomainBlockStats()**: 블록 디바이스 통계 수집
- **DomainInterfaceStats()**: 네트워크 인터페이스 통계 수집
- **DomainMemoryStats()**: 메모리 사용량 상세 정보 수집

### 2. 연결 방식

```
libvirt URI 예시:
- 로컬 시스템: qemu:///system
- 원격 시스템: qemu+ssh://user@remote-host/system
- 읽기 전용 소켓: /var/run/libvirt/libvirt-sock-ro
```

### 3. 배포 방식

#### Docker 컨테이너
```bash
docker run -d \
  --network host \
  -v /var/run/libvirt/libvirt-sock-ro:/var/run/libvirt/libvirt-sock-ro:ro \
  libvirt-exporter:latest
```

#### 시스템 서비스
```bash
libvirt-exporter --libvirt.uri="qemu:///system" --web.listen-address=":9177"
```

## 다양한 구현체

### 1. 주요 프로젝트들

#### kumina/libvirt_exporter
- 초기 구현체 중 하나 (현재 archived)
- 기본적인 libvirt 메트릭 수집
- OpenStack 메타데이터 지원 (--libvirt.export-nova-metadata 플래그)

#### zhangjianweibj/prometheus-libvirt-exporter
- OpenStack 환경에 특화된 구현체
- 상세한 OpenStack 메타데이터 지원 (instanceId, flavorName, projectName, userName 등)
- 호스트 및 도메인 정보 포함

#### Tinkoff/libvirt-exporter
- 고급 메트릭 지원 (메모리 상세 통계, vCPU 세부 정보)
- GetAllDomainStats() API 활용한 효율적 수집
- 다양한 빌드 환경 지원

### 2. 기능 비교

| 기능 | kumina | zhangjianweibj | Tinkoff |
|-----|--------|----------------|---------|
| 기본 메트릭 | ✓ | ✓ | ✓ |
| OpenStack 메타데이터 | ✓ | ✓✓ | ✓ |
| 메모리 상세 통계 | ✗ | ✗ | ✓ |
| vCPU 세부 정보 | ✗ | ✗ | ✓ |
| 풀 정보 | ✗ | ✗ | ✓ |
| 프로젝트 필터링 | ✗ | ✓ | ✗ |

## 운영 고려사항

### 1. 성능 최적화

- **캐싱**: 메트릭 수집 간격 조정을 통한 시스템 부하 최소화
- **선택적 수집**: 필요한 메트릭만 수집하여 오버헤드 감소
- **비동기 처리**: 여러 도메인의 메트릭을 병렬로 수집

### 2. 보안 고려사항

- **읽기 전용 소켓 사용**: libvirt-sock-ro 소켓을 통한 안전한 접근
- **권한 최소화**: 필요한 최소 권한만 부여
- **네트워크 보안**: 원격 연결 시 SSH 터널링 활용

### 3. 가용성 및 안정성

- **에러 처리**: 도메인 상태 변경 시 발생할 수 있는 오류 처리
- **재연결 로직**: libvirt 데몬 재시작 시 자동 재연결
- **메트릭 일관성**: 도메인 생성/삭제 시 메트릭 정합성 보장

## 활용 사례

### 1. 리소스 모니터링

- **용량 계획**: 컴퓨트 노드의 리소스 사용률 추적
- **성능 분석**: VM별 CPU, 메모리, I/O 성능 모니터링
- **이상 탐지**: 비정상적인 리소스 사용 패턴 감지

### 2. 운영 최적화

- **자동 스케일링**: 리소스 사용률 기반 자동 확장/축소
- **부하 분산**: 컴퓨트 노드 간 워크로드 분산 최적화
- **예방적 유지보수**: 성능 저하 조기 감지 및 대응

### 3. 비용 최적화

- **리소스 효율성**: 과도하게 할당된 리소스 식별
- **사용률 분석**: 프로젝트별 실제 리소스 사용량 추적
- **차지백**: 정확한 사용량 기반 비용 할당

## 결론

LIBVIRT-EXPORTER는 OpenStack Nova 컴퓨트 노드에서 실제 VM의 리소스 상태를 정확하고 효율적으로 모니터링하기 위한 필수 도구입니다. OpenStack API의 한계를 극복하고 하이퍼바이저 레벨에서의 상세한 메트릭을 제공함으로써, 클라우드 인프라의 성능 모니터링, 용량 계획, 운영 최적화에 중요한 역할을 수행합니다.

특히 대규모 OpenStack 환경에서는 API 기반 모니터링의 성능 한계를 극복하고 실시간성을 보장하기 위해 libvirt를 통한 직접적인 메트릭 수집이 필수적이며, LIBVIRT-EXPORTER는 이러한 요구사항을 효과적으로 충족하는 솔루션입니다.