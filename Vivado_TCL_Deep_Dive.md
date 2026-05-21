# Xilinx Vivado TCL 완벽 가이드 — Deep Dive 교육 자료

> **대상**: Vivado를 사용하는 FPGA 엔지니어 (중급 이상)
> **목표**: 단순 GUI 조작을 넘어 TCL 스크립팅으로 설계 흐름을 완전히 제어하고 자동화한다.
> **버전**: Vivado 2025.2 기준 (AMD)

---

## 목차

1. [TCL 기초 — Vivado에서 살아남기 위한 필수 문법](#1-tcl-기초--vivado에서-살아남기-위한-필수-문법)
2. [Vivado TCL 아키텍처](#2-vivado-tcl-아키텍처)
3. [Vivado 객체 모델 (Object Model)](#3-vivado-객체-모델-object-model)
4. [프로젝트 모드 (Project Mode)](#4-프로젝트-모드-project-mode)
5. [비프로젝트 모드 (Non-Project Mode)](#5-비프로젝트-모드-non-project-mode)
6. [주요 Vivado TCL 명령어 총정리](#6-주요-vivado-tcl-명령어-총정리)
7. [타이밍 제약 (XDC)과 분석](#7-타이밍-제약-xdc과-분석)
8. [고급 스크립팅 기법](#8-고급-스크립팅-기법)
9. [실전 종합 예제](#9-실전-종합-예제)
10. [디버깅 및 문제 해결](#10-디버깅-및-문제-해결)
11. [Appendix](#11-appendix)

---

# 1. TCL 기초 — Vivado에서 살아남기 위한 필수 문법

## 1.1 TCL이란?

TCL(Tool Command Language)은 John Ousterhout이 만든 스크립트 언어로, **모든 것이 명령어(Command)** 이다. Vivado는 TCL 해석기를 내장하고 있어, GUI에서 Tcl Console을 통해 모든 조작이 가능하다.

```
모든 것은 명령어다. (Everything is a command.)
모든 것은 문자열이다. (Everything is a string.)
```

## 1.2 명령어 구조

```tcl
명령어 인수1 인수2 인수3 ...
```

```tcl
puts "Hello, Vivado!"
# => Hello, Vivado!

set my_var "hello"
puts $my_var
# => hello
```

## 1.3 변수와 치환 (Substitution)

TCL은 세 가지 치환 방식을 갖는다:

| 치환 | 기호 | 설명 |
|------|------|------|
| 변수 치환 | `$var` | 변수 값을 가져옴 |
| 명령어 치환 | `[command]` | 명령어 실행 결과로 대체 |
| 역슬래시 치환 | `\n`, `\$` | 특수 문자 이스케이프 |

```tcl
set name "Vivado"
puts "Hello, $name"           ;# 변수 치환
puts "Date: [clock format [clock seconds]]"  ;# 명령어 치환
puts "Dollar sign: \$name"     ;# 역슬래시 치환
```

## 1.4 따옴표와 중괄호

| 구분 | 치환 수행 | 예 |
|------|-----------|-----|
| `"` 큰따옴표 | 수행됨 | `puts "Hello $name"` |
| `{` 중괄호 | 수행 안 됨 | `puts {Hello $name}` → 그대로 출력 |
| `[` 대괄호 | 명령어 치환 | `set a [expr {1 + 2}]` |

```tcl
set x 10
puts "$x + 2 = [expr {$x + 2}]"   ;# 변환 O: "10 + 2 = 12"
puts {$x + 2 = [expr {$x + 2}]}   ;# 변환 X: "$x + 2 = [expr {$x + 2}]"
```

## 1.5 리스트 (List)

TCL에서 리스트는 **공백으로 구분된 문자열**이다. Vivado에서 가장 많이 사용하는 자료구조.

```tcl
set files {top.v module_a.v module_b.v}
set files "top.v module_a.v module_b.v"   ;# 동일

llength $files       ;# => 3
lindex $files 0      ;# => top.v
lappend files "extra.v"  ;# 요소 추가
lsearch $files "top.v"   ;# => 0 (인덱스 반환, 없으면 -1)
linsert $files 1 "new.v" ;# 1번 위치에 삽입
```

## 1.6 제어문

```tcl
# 조건문
if { [get_property IS_PRIMITIVE [get_cells u_reg]] == 1 } {
    puts "Primitive cell"
} elseif { $x > 5 } {
    puts "x is greater than 5"
} else {
    puts "else"
}
```

```tcl
# 반복문
foreach cell [get_cells -hier -filter {IS_SEQUENTIAL == 1}] {
    puts [get_property REF_NAME $cell]
}

for {set i 0} {$i < 10} {incr i} {
    puts "Iteration $i"
}

while { [llength $queue] > 0 } {
    set item [lindex $queue 0]
    set queue [lrange $queue 1 end]
}
```

## 1.7 프로시저 (Procedure)

```tcl
proc reportCriticalPaths {outputFile {maxPaths 5}} {
    set fp [open $outputFile w]
    puts $fp "Slack,Startpoint,Endpoint"
    foreach path [get_timing_paths -max_paths $maxPaths -nworst 1] {
        set slack [get_property SLACK $path]
        set start [get_property STARTPOINT_PIN $path]
        set end   [get_property ENDPOINT_PIN $path]
        puts $fp "$slack,$start,$end"
    }
    close $fp
    puts "Report written to $outputFile"
}
```

## 1.8 파일 입출력

```tcl
# 파일 읽기
set fp [open "input.txt" r]
set content [read $fp]
close $fp

# 파일 쓰기
set fp [open "output.csv" w]
puts $fp "Header1,Header2,Header3"
foreach item $list {
    puts $fp "$item,1,2"
}
close $fp
```

## 1.9 정규표현식

```tcl
set cell_name "u_top/u_sub/u_reg_0"
if { [regexp {u_reg_(\d+)} $cell_name -> idx] } {
    puts "Register index: $idx"
}

# 매칭되는 모든 셀 찾기
set all_cells [get_cells -hier -regexp {.*_reg_\d+}]
```

## 1.10 Vivado TCL에서 자주 쓰는 유틸리티

```tcl
# 시간 측정
set start_time [clock seconds]
# ... 작업 수행 ...
set elapsed [expr {[clock seconds] - $start_time}]
puts "Elapsed: ${elapsed}s"

# 현재 Vivado 명령어 버전 확인
puts [version]

# Vivado 환경 변수 확인
puts "Vivado version: [info nameofexecutable]"
puts "Current dir: [pwd]"
```

---

# 2. Vivado TCL 아키텍처

## 2.1 Vivado TCL 실행 모드

Vivado는 세 가지 TCL 실행 모드를 제공한다:

| 모드 | 명령어 | 특징 |
|------|--------|------|
| **Shell 모드** | `vivado -mode tcl` | GUI 없이 TCL 셸만 실행. 가장 가벼움 |
| **Batch 모드** | `vivado -mode batch -source script.tcl` | 비대화형. 서버/CI/CD에 적합 |
| **GUI 모드** | `vivado` | GUI + Tcl Console 함께 실행 |

```powershell
# Batch 모드 실행 예시
vivado -mode batch -source run_synth.tcl -log synth.log -journal synth.jou

# Tcl Shell 모드
vivado -mode tcl

# GUI 없이 TCL 스크립트만 실행
vivado -mode tcl -source my_script.tcl
```

## 2.2 Vivado TCL 이름 공간 (Namespace)

Vivado는 자체 TCL 확장 명령어를 `::tclapp::` 네임스페이스에 구성한다:

```
::tclapp::<category>::<command>
```

```tcl
# 사용자 정의 TCL 앱 로드
source "$::tclapp_dir/mytools/report_utils.tcl"
namespace eval ::tclapp::mytools {
    # ... 사용자 정의 명령어 ...
}
```

## 2.3 Vivado TCL 성능 팁

```tcl
# 느림 - 매번 get_* 호출
foreach cell [get_cells -hier -filter {REF_NAME =~ "FD*"}] {
    puts [get_property LOC $cell]
}

# 빠름 - collect 명령어로 한 번에 객체 수집
set cells [get_cells -hier -filter {REF_NAME =~ "FD*"}]
foreach cell $cells {
    puts [get_property LOC $cell]
}

# 더 빠름 - filter를 한 번에
set props [list_property [get_cells -hier -filter {REF_NAME =~ "FD*"}]]
```

> **성능 원칙**: `get_*` 호출은 최소화하고, `filter` 표현식을 활용해 DB 내에서 필터링하라.

---

# 3. Vivado 객체 모델 (Object Model)

## 3.1 주요 객체 타입

Vivado는 설계 데이터를 객체(Object)로 관리한다:

| 객체 타입 | 설명 | 생성/조회 명령어 |
|-----------|------|-----------------|
| **cell** | 논리 셀 (인스턴스) | `get_cells` |
| **pin** | 셀의 핀 | `get_pins` |
| **net** | 신호 연결선 | `get_nets` |
| **port** | 최상위 포트 | `get_ports` |
| **clock** | 클럭 객체 | `get_clocks` |
| **timing_path** | 타이밍 경로 | `get_timing_paths` |
| **design** | 전체 설계 | `current_design` |

```tcl
# 계층적 셀 조회
get_cells -hier                       ;# 모든 셀
get_cells -hier -filter {REF_NAME =~ "FD*"}  ;# FD로 시작하는 모든 플립플롭
get_cells -hier -regexp {.*u_reg.*}   ;# 정규표현식 매칭

# 핀 조회
get_pins -hier -filter {DIRECTION == IN}
get_pins -of [get_cells u_reg]        ;# 특정 셀의 모든 핀

# 넷 조회
get_nets -hier
get_nets -of [get_pins u_reg/Q]       ;# 특정 핀에 연결된 넷
get_nets -segments [get_nets clk]     ;# 클럭 넷의 세그먼트
```

## 3.2 주요 객체 계층 구조

```
design (current_design)
 ├── port (get_ports)
 ├── cell (get_cells)
 │    ├── pin (get_pins)
 │    ├── bel (get_bels)
 │    └── site (get_sites)
 ├── net (get_nets)
 └── clock (get_clocks)
```

## 3.3 속성 조회와 설정

```tcl
# 속성 조회
get_property REF_NAME [get_cells u_reg]
get_property LOC [get_cells u_reg]

# 모든 속성 보기
report_property [get_cells u_reg]

# 속성 설정
set_property LOC SLICE_X10Y10 [get_cells u_reg]
set_property IOB TRUE [get_cells u_reg_out]
set_property IS_LOC_FIXED 1 [get_cells u_reg]

# 다중 객체에 속성 설정
set_property -all [get_cells -hier -filter {REF_NAME =~ "FD*"}] IS_LOC_FIXED 1
```

## 3.4 객체 필터링 표현식

Vivado 객체 명령어의 `-filter` 옵션은 강력한 필터링을 제공한다:

```tcl
# 연산자
# ==, !=   : 동등 비교
# =~       : glob 매칭 (와일드카드)
# !~       : glob 부정 매칭
# <, >, <=, >= : 숫자 비교
# &&, ||, !: 논리 연산

# 필터 예제
get_cells -hier -filter {IS_SEQUENTIAL == 1 && REF_NAME =~ "FD*"}
get_pins -hier -filter {DIRECTION == OUT && IS_LEAF == 1}
get_nets -hier -filter {HAS_FANOUT > 100}
get_ports -hier -filter {DIRECTION == IN}
```

## 3.5 `-of_objects` 연결 탐색

객체 간 관계를 타고 이동할 때 사용:

```tcl
# 셀 -> 핀 -> 넷 -> 핀 -> 셀
get_cells -of [get_pins -of [get_nets -of [get_pins -of [get_cells u_reg] \
    -filter {NAME =~ "*Q"}]]]
    
# 클럭이 연결된 모든 핀
get_pins -hier -of [get_clocks system_clk]

# 특정 넷에 연결된 모든 셀
get_cells -of [get_nets data_bus_*] -hier
```

---

# 4. 프로젝트 모드 (Project Mode)

## 4.1 프로젝트 모드 개요

프로젝트 모드는 Vivado IDE와 동일한 파일/설정 관리 구조를 TCL로 제어한다. 소스 관리, 빌드 추적, 설정 저장에 유리하다.

## 4.2 프로젝트 생성 흐름

```tcl
# 1. 프로젝트 생성
create_project my_project ./my_project -part xc7k325tfbg900-2
# 또는 기존 프로젝트 열기
open_project ./my_project/my_project.xpr

# 2. 소스 파일 추가
add_files -norecurse {
    ./src/top.v
    ./src/module_a.v
    ./src/module_b.vhd
}

# 3. IP 추가
read_ip ./ip/clk_wiz_0/clk_wiz_0.xci
upgrade_ip [get_ips clk_wiz_0]
generate_ip -force [get_ips clk_wiz_0]

# 4. 블록 디자인 (BD) 추가
read_bd ./bd/design_1/design_1.bd
generate_target all [get_files design_1.bd]

# 5. constraint 파일 추가
add_files -fileset constrs_1 ./constraints/timing.xdc
set_property target_constrs_file [get_files timing.xdc] [current_fileset -constrset]

# 6. 최상위 모듈 지정
set_property top top_module [current_fileset]
```

## 4.3 합성/구현/비트스트림 생성

```tcl
# 합성
launch_runs synth_1 -jobs 4
wait_on_run synth_1

# 합성 결과 열기 (보고서 생성을 위해)
open_run synth_1
report_timing_summary -file ./reports/post_synth_timing.rpt
report_utilization -file ./reports/post_synth_util.rpt
close_design

# 구현 (place & route)
launch_runs impl_1 -to_step route_design -jobs 4
wait_on_run impl_1

# 구현 결과 열기
open_run impl_1
report_timing_summary -file ./reports/post_route_timing.rpt
report_utilization -file ./reports/post_route_util.rpt
report_power -file ./reports/post_route_power.rpt

# 비트스트림 생성
launch_runs impl_1 -to_step write_bitstream
wait_on_run impl_1
```

## 4.4 프로젝트 모드 주요 명령어 표

| 명령어 | 설명 |
|--------|------|
| `create_project` | 새 프로젝트 생성 |
| `open_project` | 기존 .xpr 프로젝트 열기 |
| `close_project` | 프로젝트 닫기 |
| `add_files` | 소스/컨스트레인트 파일 추가 |
| `remove_files` | 파일 제거 |
| `import_files` | 프로젝트 디렉토리로 파일 복사 후 추가 |
| `set_property` | 프로젝트/파일 속성 설정 |
| `launch_runs` | 합성/구현/비트스트림 실행 |
| `wait_on_run` | 실행 완료까지 대기 |
| `open_run` | 합성/구현 결과를 메모리에 로드 |
| `close_design` | 열린 설계 데이터 닫기 |
| `reset_run` | 실행 결과 초기화 |
| `delete_runs` | 실행 설정 삭제 |
| `current_fileset` | 현재 파일셋获取 |
| `get_files` | 프로젝트 파일 조회 |
| `get_runs` | 실행 목록 조회 |
| `get_reports` | 생성된 리포트 조회 |

---

# 5. 비프로젝트 모드 (Non-Project Mode)

## 5.1 비프로젝트 모드 개요

비프로젝트 모드는 **메모리 기반(in-memory)** 설계 흐름이다. 프로젝트 파일(.xpr)을 생성하지 않으며, 모든 조작이 TCL 명령어로 직접 이루어진다. CI/CD, 자동화, 고급 스크립팅에 적합하다.

**장점**:
- 프로젝트 관리 오버헤드 없음
- 완전한 자동화 가능
- 단계별 세밀한 제어
- 버전 관리에 용이 (스크립트만 관리)

**단점**:
- 설계 상태를 직접 저장/복원해야 함
- GUI와의 통합이 제한적
- 결과 재현을 위해 스크립트를 완전히 관리해야 함

## 5.2 전체 비프로젝트 흐름

```tcl
###############################################################################
# 완전한 비프로젝트 모드 TCL 스크립트 예제
###############################################################################

# 설정
set outputDir ./output
set partName xc7k325tfbg900-2
set topName top

file mkdir $outputDir

# STEP 1: HDL 파일 읽기
read_verilog [glob ./src/*.v]
read_vhdl [glob ./src/*.vhd]

# STEP 2: IP 읽기 및 생성
read_ip ./ip/clk_gen/clk_gen.xci
generate_target all [get_ips clk_gen]

# STEP 3: 링크 (엘라보레이션)
link_design -part $partName -top $topName

# STEP 4: 컨스트레인트 읽기 (link_design 이후!)
read_xdc ./constraints/timing.xdc

# STEP 5: 합성
synth_design -top $topName -part $partName
write_checkpoint -force $outputDir/post_synth.dcp
report_timing_summary -file $outputDir/post_synth_timing.rpt
report_utilization  -file $outputDir/post_synth_util.rpt

# STEP 6: 최적화
opt_design
write_checkpoint -force $outputDir/post_opt.dcp
report_timing_summary -file $outputDir/post_opt_timing.rpt

# STEP 7: 배치 (Placement)
place_design
write_checkpoint -force $outputDir/post_place.dcp
report_timing_summary -file $outputDir/post_place_timing.rpt
report_utilization  -file $outputDir/post_place_util.rpt

# 배치 후 타이밍이 나쁘면 Physical Opt.
if { [get_property SLACK [get_timing_paths -max_paths 1 -nworst 1]] < 0 } {
    puts "Post-Place timing violation detected, running phys_opt_design..."
    phys_opt_design
    write_checkpoint -force $outputDir/post_physopt.dcp
}

# STEP 8: 라우팅 (Routing)
route_design
write_checkpoint -force $outputDir/post_route.dcp
report_timing_summary -file $outputDir/post_route_timing.rpt
report_utilization   -file $outputDir/post_route_util.rpt
report_power         -file $outputDir/post_route_power.rpt
report_drc           -file $outputDir/post_route_drc.rpt

# STEP 9: 비트스트림 생성
write_bitstream -force $outputDir/top.bit

puts "=== Design flow completed successfully ==="
```

## 5.3 `synth_design` 주요 옵션

```tcl
# 기본 합성
synth_design -top top -part xc7k325tfbg900-2

# 고급 합성 옵션
synth_design -top top \
    -part xc7k325tfbg900-2 \
    -flatten_hierarchy rebuilt \    ;# rebuilt | full | none
    -gated_clock_conversion auto \   ;# off | on | auto
    -directive AreaOptimized_high \  ;# Default | AreaOptimized_high | RuntimeOptimized | ...
    -fsm_extraction one_hot \        ;# auto | one_hot | binary | gray | ...
    -keep_equivalent_registers \
    -resource_sharing auto \         ;# auto | off | on
    -no_lc \                         ;# LUT 결합 비활성화
    -shreg_min_size 3 \              ;# SRL 최소 길이
    -cascade_dsp auto \              ;# DSP 캐스케이드
    -control_set_opt_threshold auto  ;# 제어 세트 최적화 임계값
    -max_bram -1 \                   ;# 최대 BRAM (-1 = 제한 없음)
    -max_dsp -1 \                    ;# 최대 DSP
    -verbose
```

## 5.4 `place_design` 및 `route_design` 디렉티브

```tcl
# Place 옵션
place_design                                    ;# Default 디렉티브
place_design -directive ExtraPostPlacementOpt    ;# 배치 후 추가 최적화
place_design -directive AltSpreadLogic_high      ;# 로직 분산 최적화
place_design -directive WLDrivenBlockPlacement   ;# Wire-Length 중심 배치
place_design -directive EarlyBlockPlacement      ;# 빠른 배치 (대략적)

# Route 옵션
route_design                                    ;# Default
route_design -directive Explore                  ;# 다양한 라우팅 시도
route_design -directive NoTimingRelaxation        ;# 타이밍 완화 없음
route_design -directive MoreGlobalIterations      ;# 글로벌 반복 증가
route_design -directive HigherDelayCost           ;# 지연 비용 가중치 증가
route_design -directive AdvancedSkewModeling      ;# 스큐 모델링 고급 모드
route_design -directive AggressiveExplore         ;# 가장 공격적인 탐색
```

## 5.5 증분(Incremental) 플로우

```tcl
# 이전 구현 결과를 참조(reference)로 사용
set_property STEPS.PHYS_OPT_DESIGN.IS_ENABLED true [get_runs impl_1]
set_property STEPS.ROUTE_DESIGN.ARGS.DIRECTIVE Explore [get_runs impl_1]

# Reference checkpoint 사용
set_property RD.PREVIOUS_REFERENCE_CHECKPOINT ./prev_impl/post_route.dcp [get_runs impl_1]
launch_runs impl_1
```

---

# 6. 주요 Vivado TCL 명령어 총정리

## 6.1 파일 I/O

| 명령어 | 설명 |
|--------|------|
| `read_verilog <files>` | Verilog 파일 읽기 |
| `read_vhdl <files>` | VHDL 파일 읽기 |
| `read_edif <file>` | EDIF 넷리스트 읽기 |
| `read_ip <file>` | IP (XCI) 읽기 |
| `read_bd <file>` | 블록 디자인 (BD) 읽기 |
| `read_xdc <file>` | XDC 제약 파일 읽기 |
| `read_checkpoint <dcp>` | DCP 체크포인트 읽기 |
| `write_checkpoint <dcp>` | DCP 체크포인트 저장 |
| `write_bitstream <bit>` | 비트스트림 생성 |
| `write_verilog <file>` | 넷리스트를 Verilog로 추출 |
| `write_vhdl <file>` | 넷리스트를 VHDL로 추출 |
| `write_xdc <file>` | 현재 제약을 XDC로 추출 |

## 6.2 설계 흐름

| 명령어 | 설명 |
|--------|------|
| `link_design` | 엘라보레이션 수행 |
| `synth_design` | 논리 합성 |
| `opt_design` | 논리 최적화 |
| `place_design` | 배치 (Placement) |
| `phys_opt_design` | 물리 최적화 |
| `route_design` | 배선 (Routing) |
| `report_timing_summary` | 타이밍 요약 리포트 |
| `report_utilization` | 자원 사용률 리포트 |
| `report_power` | 전력 분석 리포트 |
| `report_drc` | DRC 리포트 |
| `report_clock_interaction` | 클럭 간 상호작용 리포트 |
| `report_methodology` | 방법론 체크 |

## 6.3 객체 쿼리

| 명령어 | 설명 |
|--------|------|
| `get_cells [-hier] [-filter <expr>] [-regexp] [-of <obj>]` | 셀 조회 |
| `get_pins [-hier] [-filter <expr>] [-of <obj>]` | 핀 조회 |
| `get_nets [-hier] [-filter <expr>] [-of <obj>] [-segments]` | 넷 조회 |
| `get_ports [-filter <expr>]` | 포트 조회 |
| `get_clocks [-filter <expr>] [-of <obj>]` | 클럭 조회 |
| `get_timing_paths [-from <pins>] [-to <pins>] [-max_paths <n>]` | 타이밍 경로 조회 |
| `get_bels [-filter <expr>] [-of <obj>]` | BEL 조회 |
| `get_sites [-filter <expr>]` | 사이트 조회 |
| `get_pblocks` | Pblock 조회 |
| `get_designs` | 현재 디자인 조회 |

## 6.4 객체 속성

| 명령어 | 설명 |
|--------|------|
| `get_property <name> <object>` | 속성값 읽기 |
| `set_property <name> <value> <objects>` | 속성값 설정 |
| `report_property <object>` | 모든 속성 출력 |
| `list_property <object>` | 속성 이름 목록 반환 |
| `list_property_value <name> <type>` | 속성의 가능한 값 목록 |

## 6.5 타이밍 제약

| 명령어 | 설명 |
|--------|------|
| `create_clock -period <period> -name <name> [get_ports <port>]` | 클럭 생성 |
| `create_generated_clock -source <src> -divide_by <n> <target>` | 생성 클럭 정의 |
| `set_input_delay -clock <clk> <delay> [get_ports <ports>]` | 입력 지연 설정 |
| `set_output_delay -clock <clk> <delay> [get_ports <ports>]` | 출력 지연 설정 |
| `set_max_delay <delay> [-from <pins>] [-to <pins>]` | 최대 지연 제약 |
| `set_min_delay <delay> [-from <pins>] [-to <pins>]` | 최소 지연 제약 |
| `set_false_path [-from <pins>] [-to <pins>]` | false path 설정 |
| `set_multicycle_path -setup <n> [-hold <m>] <paths>` | 멀티사이클 경로 |
| `set_clock_groups -asynchronous -group <clk1> -group <clk2>` | 비동기 클럭 그룹 |

## 6.6 디자인 규칙

| 명령어 | 설명 |
|--------|------|
| `current_design` | 현재 활성 설계 반환 |
| `list_cmd` | 사용 가능한 TCL 명령어 목록 |
| `help <command>` | 명령어 도움말 |
| `history` | 명령어 히스토리 |
| `source <script>` | TCL 스크립트 실행 |
| `stop_gui` | GUI 종료 |
| `start_gui` | GUI 시작 |

---

# 7. 타이밍 제약 (XDC)과 분석

## 7.1 XDC 파일의 본질

**XDC = Xilinx Design Constraints** = TCL 명령어의 집합.

```tcl
# XDC는 TCL이다. Vivado가 부팅될 때 TCL 인터프리터가 XDC를 해석한다.
# 따라서 조건문, 변수, 프로시저를 XDC에서 사용할 수 있다.

set period 10.0
if { $period > 5.0 } {
    create_clock -period $period -name sys_clk [get_ports clk_in]
}
```

## 7.2 클럭 정의

```tcl
# 기본 클럭 정의
create_clock -period 10.000 -name clk_sys [get_ports clk_p]
create_clock -period 10.000 -name clk_sys_n -add [get_ports clk_n]

# Generated clock (MMCM/PLL 출력)
create_generated_clock -name clk_mmcm_out \
    -source [get_pints mmcm_inst/CLKIN1] \
    -multiply_by 4 -divide_by 5 \
    [get_pins mmcm_inst/CLKOUT1]

# 가상 클럭 (입력/출력 포트 기준용)
create_clock -period 5.000 -name virtual_clk
```

## 7.3 I/O Delays

```tcl
# 입력 delay
set_input_delay -clock clk_sys -max 3.0 [get_ports data_in]
set_input_delay -clock clk_sys -min 1.5 [get_ports data_in]

# 출력 delay
set_output_delay -clock clk_sys -max 4.5 [get_ports data_out] -reference_pin [get_pins obuf_inst/O]
set_output_delay -clock clk_sys -min 0.5 [get_ports data_out]
```

## 7.4 타이밍 예외

```tcl
# False path
set_false_path -from [get_clocks rst_clk] -to [get_clocks sys_clk]
set_false_path -through [get_pins u_sync/*/Q]

# Multicycle path - 2 클럭 주기 허용
set_multicycle_path -setup 2 -from [get_cells u_slow_logic] -to [get_cells u_fast_reg]
set_multicycle_path -hold 1 -from [get_cells u_slow_logic] -to [get_cells u_fast_reg]

# Max delay (비동기 경로)
set_max_delay 4.0 -from [get_cells u_async_src] -to [get_cells u_async_dst]
set_max_delay -datapath_only 3.0 -from [get_pins u_reg_a/C] -to [get_pins u_reg_b/D]
```

## 7.5 타이밍 분석 명령어

```tcl
# 타이밍 요약 리포트
report_timing_summary -delay_type min_max -report_unconstrained -check_timing_verbose

# 상세 타이밍 경로
report_timing -from [get_pins u_reg_a/C] -to [get_pins u_reg_b/D] -delay_type max

# 특정 조건에 맞는 경로
report_timing -max_paths 100 -nworst 5 -slack_lesser_than 0 -sort_by slack

# 클럭 상호작용
report_clock_interaction -delay_type min_max

# 타이밍 경로를 TCL 객체로 얻어서 커스텀 분석
set paths [get_timing_paths -max_paths 10 -nworst 1 -slack_lesser_than 0]
foreach path $paths {
    set slack      [get_property SLACK $path]
    set start_pin  [get_property STARTPOINT_PIN $path]
    set end_pin    [get_property ENDPOINT_PIN $path]
    set path_delay [get_property DATAPATH_DELAY $path]
    set clock_skew [get_property CLOCK_SKEW $path]
    puts "$slack : $start_pin -> $end_pin (datapath=$path_delay, skew=$clock_skew)"
}
```

## 7.6 타이밍 객체 속성

`get_timing_paths`로 얻은 경로 객체의 주요 속성:

| 속성 | 설명 |
|------|------|
| `SLACK` | 타이밍 여유 |
| `STARTPOINT_PIN` | 시작 핀 |
| `ENDPOINT_PIN` | 종료 핀 |
| `DATAPATH_DELAY` | 데이터 경로 지연 |
| `CLOCK_SKEW` | 클럭 스큐 |
| `REQUIRED_TIME` | 요구 시간 |
| `ARRIVAL_TIME` | 도달 시간 |
| `LOGIC_LEVELS` | 로직 레벨 수 |
| `PATH_TYPE` | 경로 유형 (setup/hold) |
| `GROUP` | 경로 그룹 |
| `IS_SUBPATH` | 서브경로 여부 |

---

# 8. 고급 스크립팅 기법

## 8.1 Hook Script — 자동화 포인트

Vivado는 합성/구현 각 단계 전후에 사용자 스크립트를 자동 실행하는 Hook을 지원한다.

```
launch_runs impl_1
                │
        ┌───────┴───────┐
        │ tcl.pre        │  ← 구현 전 Hook
        │ opt_design     │
        │ tcl.post       │  ← 최적화 후 Hook
        │ place_design   │
        │ tcl.post       │  ← 배치 후 Hook
        │ phys_opt_design│
        │ tcl.post       │  ← 물리 최적화 후 Hook
        │ route_design   │
        │ tcl.post       │  ← 라우팅 후 Hook
        └───────────────┘
```

```tcl
# Hook 스크립트 설정 (프로젝트 모드)
set_property STEPS.SYNTH_DESIGN.TCL.PRE  ./hooks/pre_synth.tcl  [get_runs synth_1]
set_property STEPS.SYNTH_DESIGN.TCL.POST ./hooks/post_synth.tcl [get_runs synth_1]

set_property STEPS.OPT_DESIGN.TCL.POST    ./hooks/post_opt.tcl    [get_runs impl_1]
set_property STEPS.PLACE_DESIGN.TCL.POST  ./hooks/post_place.tcl  [get_runs impl_1]
set_property STEPS.ROUTE_DESIGN.TCL.POST  ./hooks/post_route.tcl  [get_runs impl_1]
```

**post_route.tcl 예제 — 라우팅 완료 후 자동 분석**:
```tcl
# post_route.tcl
set outputDir [pwd]/reports
file mkdir $outputDir

# 타이밍 리포트
report_timing_summary -file $outputDir/post_route_timing.rpt -delay_type min_max

# 위반 경로 상세 분석
set viol_paths [get_timing_paths -max_paths 50 -nworst 3 -slack_lesser_than 0]
if { [llength $viol_paths] > 0 } {
    set fp [open $outputDir/violation_paths.csv w]
    puts $fp "Slack,Startpoint,Endpoint,LogicLevels,DataPathDelay,ClockSkew,Group"
    foreach path $viol_paths {
        puts $fp "[get_property SLACK $path],[get_property STARTPOINT_PIN $path],[get_property ENDPOINT_PIN $path],[get_property LOGIC_LEVELS $path],[get_property DATAPATH_DELAY $path],[get_property CLOCK_SKEW $path],[get_property GROUP $path]"
    }
    close $fp
    puts "WARNING: [llength $viol_paths] timing violations found!"
} else {
    puts "INFO: No timing violations."
}

# DRC 리포트
report_drc -file $outputDir/post_route_drc.rpt
```

## 8.2 전략 매개변수화 (Strategy Parameterization)

```tcl
# 전략을 TCL 리스트로 관리
set strategies {
    {Default       {Default           Default           Default}}
    {Explore       {Explore           Explore           Explore}}
    {Area         {AreaOptimized_high Default           Default}}
    {Quick        {RuntimeOptimized   EarlyBlockPlacement NoTimingRelaxation}}
}

set outputDir ./results
file mkdir $outputDir

foreach {name synthDir placeDir routeDir} $strategies {
    puts "Running strategy: $name"
    
    # 프로젝트 복제 또는 새로 생성
    create_project "proj_${name}" "./proj_${name}" -part xc7k325tfbg900-2 -force
    add_files [glob ./src/*.v ./src/*.vhd]
    add_files -fileset constrs_1 ./constraints/timing.xdc
    set_property top top [current_fileset]
    
    # 합성
    synth_design -top top -part xc7k325tfbg900-2 -directive $synthDir
    write_checkpoint -force $outputDir/${name}_synth.dcp
    report_timing_summary -file $outputDir/${name}_synth_timing.rpt
    
    # 최적화
    opt_design
    write_checkpoint -force $outputDir/${name}_opt.dcp
    
    # 배치
    place_design -directive $placeDir
    write_checkpoint -force $outputDir/${name}_place.dcp
    
    # 라우팅
    route_design -directive $routeDir
    write_checkpoint -force $outputDir/${name}_route.dcp
    
    # 결과
    report_timing_summary -file $outputDir/${name}_final_timing.rpt
    report_utilization -file $outputDir/${name}_final_util.rpt
    
    set slack [get_property SLACK [get_timing_paths -max_paths 1 -nworst 1]]
    puts "  -> Slack: ${slack}ns"
    
    close_project
}
```

## 8.3 병렬 실행 및 배치 작업

```tcl
# 여러 구현을 병렬로 실행
set strategies {Default Explore AggressiveExplore}
set run_ids {}

foreach strategy $strategies {
    set run_name "impl_${strategy}"
    create_run $run_name -parent_run synth_1
    set_property STEPS.ROUTE_DESIGN.ARGS.DIRECTIVE $strategy [get_runs $run_name]
    
    # 비동기 실행
    launch_runs $run_name -remote localhost:0 &
    lappend run_ids $run_name
}

# 모든 run 완료 대기
foreach run_id $run_ids {
    wait_on_run $run_id
    open_run $run_id
    report_timing_summary -file ./reports/${run_id}_timing.rpt
    close_design
    puts "$run_id completed."
}
```

## 8.4 DCP 기반 분석 자동화

```tcl
# 여러 DCP에 대해 일괄 분석
proc analyze_dcp {dcp_file output_dir} {
    file mkdir $output_dir
    set base [file rootname [file tail $dcp_file]]
    
    open_checkpoint $dcp_file
    
    # 타이밍
    report_timing_summary -file "${output_dir}/${base}_timing.rpt"
    
    # 자원
    report_utilization -file "${output_dir}/${base}_util.rpt" -hierarchical
    
    # 전력
    report_power -file "${output_dir}/${base}_power.rpt"
    
    # 클럭
    report_clock_interaction -file "${output_dir}/${base}_clock.rpt"
    
    # 타이밍 위반이 있는지 체크
    set viol [get_timing_paths -max_paths 1 -slack_lesser_than 0]
    if { [llength $viol] > 0 } {
        set slack [get_property SLACK [lindex $viol 0]]
        puts "*** WARNING: $base has timing violations (Worst slack: ${slack}ns)"
    } else {
        puts "OK: $base - no timing violations"
    }
    
    close_design
}

foreach dcp [glob ./checkpoints/*.dcp] {
    analyze_dcp $dcp ./analysis_results
}
```

## 8.5 Vivado TCL 앱 및 패키징

```tcl
# Vivado TCL 앱 구조
#
# $tclapp_dir/myutils/
#   ├── myutils.tcl       (메인 모듈)
#   └── pkgIndex.tcl      (패키지 인덱스)

# myutils.tcl
namespace eval ::tclapp::myutils {
    namespace export reportWorstPaths checkTimingVsUtil generateSummary
}

proc ::tclapp::myutils::reportWorstPaths {{num_paths 10}} {
    set paths [get_timing_paths -max_paths $num_paths -nworst 1 -sort_by slack]
    set idx 0
    foreach path $paths {
        puts "#[incr idx]: Slack=[get_property SLACK $path]  \
            [get_property STARTPOINT_PIN $path] -> \
            [get_property ENDPOINT_PIN $path]"
    }
}

proc ::tclapp::myutils::checkTimingVsUtil {} {
    set slack [get_property SLACK [get_timing_paths -max_paths 1 -nworst 1]]
    set util [get_property UTILIZATION [current_design]]
    puts "Slack: $slack, Utilization: $util"
}

# 사용법
# source $tclapp_dir/myutils/myutils.tcl
# ::tclapp::myutils::reportWorstPaths
```

---

# 9. 실전 종합 예제

## 9.1 완전 자동화 빌드 스크립트

```tcl
#!/usr/bin/env vivado -mode batch -source
###############################################################################
# build.tcl — 완전 자동화된 Vivado 빌드 스크립트
# 사용법: vivado -mode batch -source build.tcl -tclargs --part xc7k325tfbg900-2 --top top
###############################################################################

# 명령행 인수 파싱
array set args {
    -part    xc7k325tfbg900-2
    -top     top
    -jobs    4
    -output  ./build_output
}
for {set i 0} {$i < [llength $::argv]} {incr i} {
    set arg [lindex $::argv $i]
    if { [info exists args($arg)] } {
        set args($arg) [lindex $::argv [incr i]]
    }
}

set outputDir $args(-output)
file mkdir $outputDir
set logFile "${outputDir}/build.log"
proc log {msg} {
    global logFile
    set timestamp [clock format [clock seconds] -format "%Y-%m-%d %H:%M:%S"]
    puts "$timestamp: $msg"
    if { [catch {set fp [open $logFile a]}] == 0 } {
        puts $fp "$timestamp: $msg"
        close $fp
    }
}

log "=== Build started ==="
log "Part: $args(-part), Top: $args(-top), Jobs: $args(-jobs)"

# STEP 1: HDL 파일 읽기
log "Reading design files..."
if { [catch {
    read_verilog [glob ./src/hdl/*.v ./src/hdl/*.sv]
    read_vhdl   [glob ./src/hdl/*.vhd]
    read_ip     [glob ./src/ip/*.xci]
} err] } {
    log "ERROR: Failed to read design files: $err"
    exit 1
}

# STEP 2: IP 생성
log "Generating IP..."
foreach ip [get_ips] {
    generate_target all [get_ips $ip]
    log "  IP $ip generated"
}

# STEP 3: 링크
log "Linking design..."
if { [catch {link_design -part $args(-part) -top $args(-top)} err] } {
    log "ERROR: link_design failed: $err"
    exit 1
}

# STEP 4: 컨스트레인트
log "Reading constraints..."
if { [catch {read_xdc [glob ./src/xdc/*.xdc]} err] } {
    log "WARNING: No constraints found or error: $err"
}

# STEP 5: 합성
log "Running synthesis..."
if { [catch {
    synth_design -top $args(-top) -part $args(-part) -directive AreaOptimized_high
    write_checkpoint -force "${outputDir}/post_synth.dcp"
    report_timing_summary -file "${outputDir}/post_synth_timing.rpt"
    report_utilization  -file "${outputDir}/post_synth_util.rpt"
} err] } {
    log "ERROR: Synthesis failed: $err"
    exit 1
}

# STEP 6: 구현
log "Running implementation..."
foreach {step stepName} {opt_design "Optimization" place_design "Placement" route_design "Routing"} {
    log "  $stepName..."
    if { [catch {$step} err] } {
        log "ERROR: $stepName failed: $err"
        exit 1
    }
    set step_short [string range $step 0 4]
    write_checkpoint -force "${outputDir}/post_${step_short}.dcp"
}

# STEP 7: 최종 분석
log "Generating final reports..."
report_timing_summary -file "${outputDir}/final_timing.rpt" -delay_type min_max
report_utilization   -file "${outputDir}/final_util.rpt" -hierarchical
report_power         -file "${outputDir}/final_power.rpt"
report_drc           -file "${outputDir}/final_drc.rpt"

# STEP 8: 타이밍 체크
set viol_count [llength [get_timing_paths -max_paths 100 -slack_lesser_than 0]]
if { $viol_count > 0 } {
    log "WARNING: $viol_count timing violations found!"
    report_timing -max_paths 20 -nworst 1 -slack_lesser_than 0 -file "${outputDir}/worst_violations.rpt"
} else {
    log "INFO: No timing violations."
}

# STEP 9: 비트스트림
log "Writing bitstream..."
if { [catch {write_bitstream -force "${outputDir}/$args(-top).bit"} err] } {
    log "ERROR: Bitstream generation failed: $err"
    exit 1
}

log "=== Build completed successfully ==="
log "Output directory: $outputDir"
exit 0
```

## 9.2 타이밍 분석 대시보드 생성

```tcl
# timing_dashboard.tcl
# 여러 DCP를 분석하여 HTML 대시보드 생성

proc generate_html_report {dcp_list output_html} {
    set html "<html><head><title>Timing Dashboard</title>\n"
    append html "<style>body{font-family:Arial} table{border-collapse:collapse;width:100%} th,td{border:1px solid #ccc;padding:8px} th{background:#4472C4;color:white} .pass{color:green} .fail{color:red;font-weight:bold}</style>\n"
    append html "</head><body>"
    append html "<h1>Vivado Timing Dashboard</h1>"
    append html "<p>Generated: [clock format [clock seconds]]</p>"
    append html "<table><tr><th>Design</th><th>Worst Slack</th><th>WNS</th><th>TNS</th><th>Violations</th><th>Logic Levels</th><th>Total Paths</th></tr>"
    
    foreach dcp $dcp_list {
        set base [file rootname [file tail $dcp]]
        open_checkpoint $dcp
        
        # 메트릭 수집
        set all_paths [get_timing_paths -max_paths 10000]
        set total_paths [llength $all_paths]
        set worst_paths [get_timing_paths -max_paths 100 -nworst 1 -slack_lesser_than 999]
        
        set wns "-"
        set tns 0
        set viol_count 0
        set max_logic 0
        
        if { [llength $worst_paths] > 0 } {
            set wns [format "%.3f" [get_property SLACK [lindex $worst_paths 0]]]
            foreach p $worst_paths {
                set s [get_property SLACK $p]
                if { $s < 0 } {
                    incr viol_count
                    set tns [expr {$tns + $s}]
                }
                set ll [get_property LOGIC_LEVELS $p]
                if { $ll > $max_logic } { set max_logic $ll }
            }
            set tns [format "%.3f" $tns]
        }
        
        if { $wns >= 0 } {
            set slack_display "<span class='pass'>${wns}</span>"
        } else {
            set slack_display "<span class='fail'>${wns}</span>"
        }
        
        append html "<tr><td>$base</td><td>$slack_display</td><td>$wns</td><td>$tns</td><td>$viol_count</td><td>$max_logic</td><td>$total_paths</td></tr>"
        close_design
    }
    
    append html "</table></body></html>"
    
    set fp [open $output_html w]
    puts $fp $html
    close $fp
    puts "Dashboard written to $output_html"
}

generate_html_report [glob ./checkpoints/*.dcp] ./timing_dashboard.html
```

## 9.3 자동 FMAX 유틸리티

```tcl
proc findFmax {dcp_file {start_period 5.0} {step -0.5} {min_period 1.0}} {
    # 주어진 DCP에서 달성 가능한 최대 주파수 탐색
    open_checkpoint $dcp_file
    
    set period $start_period
    set results {}
    
    while { $period >= $min_period } {
        # 현재 클럭을 새 period로 덮어쓰기
        set freq [expr {1000.0 / $period}]  ;# MHz
        puts -nonewline "Testing Fmax = ${freq} MHz (period = ${period}ns)... "
        flush stdout
        
        # 타이밍 재분석
        set paths [get_timing_paths -max_paths 1 -nworst 1]
        if { [llength $paths] == 0 } {
            puts "FAIL (no paths)"
            break
        }
        
        set slack [get_property SLACK [lindex $paths 0]]
        
        if { $slack >= 0 } {
            puts "PASS (slack = ${slack}ns)"
            set meeting [list $period $freq $slack]
        } else {
            puts "FAIL (slack = ${slack}ns)"
            lappend results $meeting
            break
        }
        
        lappend results [list $period $freq $slack]
        set period [expr {$period + $step}]
        if { $period < $min_period } {
            set period $min_period
        }
    }
    
    close_design
    
    # 결과 출력
    puts "\n=== FMAX Scan Results ==="
    puts "Period(ns)\tFreq(MHz)\tSlack(ns)"
    foreach r $results {
        puts "[lindex $r 0]\t\t[lindex $r 1]\t\t[lindex $r 2]"
    }
    
    set best [lindex $results end]
    puts "\nRecommended FMAX: [lindex $best 1] MHz @ [lindex $best 0]ns"
    return $results
}

# 사용법
# findFmax ./checkpoints/post_route.dcp 5.0 -0.25 2.0
```

---

# 10. 디버깅 및 문제 해결

## 10.1 일반적인 오류와 해결

| 오류 | 원인 | 해결 |
|------|------|------|
| `ERROR: value exceeds bounds` | 잘못된 속성값 | `list_property_value`로 유효값 확인 |
| `ERROR: get_property needs an object` | 객체 목록이 비어 있음 | `-quiet` 옵션 또는 `llength` 체크 |
| `ERROR: set_property requires 1 object` | 다중 객체 전달 | `-all` 플래그 추가 |
| `ERROR: couldn't open file` | 경로 문제 | 절대 경로 사용 또는 `pwd` 확인 |
| `ERROR: design is not open` | `open_run` 필요 | 분석 전 `open_run synth_1` 또는 `open_checkpoint` |

## 10.2 디버깅 패턴

```tcl
# 패턴 1: 객체 존재 확인
set cells [get_cells -hier -filter {REF_NAME =~ "FD*"} -quiet]
if { [llength $cells] == 0 } {
    puts "WARNING: No FD cells found. Check if design is synthesized."
    return
}

# 패턴 2: 단계별 로깅
proc safe_eval {cmd description} {
    puts "--- Running: $description ---"
    if { [catch {eval $cmd} err] } {
        puts "ERROR: $description failed: $err"
        return -code error $err
    }
    puts "OK: $description"
}

safe_eval {synth_design -top top -part xc7k325tfbg900-2} "Synthesis"

# 패턴 3: 시간 측정 로깅
proc timed_eval {cmd description} {
    set start [clock milliseconds]
    if { [catch {eval $cmd} err] } {
        set elapsed [expr {[clock milliseconds] - $start}]
        puts "FAIL ($elapsed ms): $description"
        return -code error $err
    }
    set elapsed [expr {[clock milliseconds] - $start}]
    puts "OK ($elapsed ms): $description"
}

# 패턴 4: 중간 검증
proc verify_timing {} {
    set paths [get_timing_paths -max_paths 1 -nworst 1]
    if { [llength $paths] == 0 } {
        puts "WARNING: No timing paths found. Design may be empty."
        return 0
    }
    set slack [get_property SLACK [lindex $paths 0]]
    puts "Current worst slack: ${slack}ns"
    return [expr {$slack >= 0}]
}
```

## 10.3 Vivado 로그 분석

```tcl
# vivado.log에서 주요 정보 추출
proc parse_vivado_log {log_file} {
    set fp [open $log_file r]
    set content [read $fp]
    close $fp
    
    # 주요 섹션 추출
    set report_sections {}
    
    # 합성 결과
    if { [regexp {Synthesis finished} $content] } {
        puts "Synthesis: COMPLETED"
    }
    
    # 구현 결과
    if { [regexp {Implementation finished} $content] } {
        puts "Implementation: COMPLETED"
    }
    
    # 타이밍 위반
    set viol_count 0
    foreach line [split $content \n] {
        if { [regexp {slack.*-(\d+\.\d+)} $line -> slack] } {
            incr viol_count
            puts "  Violation #$viol_count: slack = -${slack}"
        }
    }
    
    puts "Total timing violations: $viol_count"
}
```

---

# 11. Appendix

## A. 버전별 달라진 점 (Vivado 2024.x → 2025.x)

| 변경 사항 | 설명 |
|----------|------|
| `report_compile_order` | 추가됨 — 컴파일 순서 분석 |
| `-directive AggressiveExplore` | route_design에 추가 |
| `phys_opt_design -directive Explore` | 새로운 물리 최적화 디렉티브 |
| `get_timing_paths -inter_slr` | SLR 간 경로 분석 강화 |
| `write_bitstream -encrypt` | 암호화 옵션 확장 |
| Tcl interpreter | Tcl 8.6 업데이트 (성능 향상) |

## B. 자주 사용하는 유틸리티 스니펫

```tcl
# 모든 플립플롭 개수 세기
puts [llength [get_cells -hier -filter {IS_SEQUENTIAL == 1}]]

# LUT 사용량
puts [llength [get_cells -hier -filter {PRIMITIVE_GROUP == "LUT"}]]

# 특정 경로 그룹의 최악 Slack
puts [get_property SLACK [get_timing_paths -group {clk_sys} -max_paths 1]]

# 현재 라우팅 완료율
report_route_status -of [current_design]

# 핀 카운트
puts "IO pins: [llength [get_ports -filter {DIRECTION == IN}]]"
puts "Output pins: [llength [get_ports -filter {DIRECTION == OUT}]]"

# 모듈별 계층 구조
report_hierarchy -file hierarchy.rpt

# 체크포인트 파일 정보
report_checkpoint -file checkpoint_info.rpt
```

## C. 주요 참고 문서

| 문서 번호 | 제목 |
|-----------|------|
| **UG894** | Vivado Design Suite User Guide: Using Tcl Scripting |
| **UG835** | Vivado Design Suite Tcl Command Reference Guide |
| **UG892** | Vivado Design Flows Overview |
| **UG901** | Vivado Synthesis |
| **UG904** | Vivado Implementation |
| **UG903** | Vivado Using Constraints |
| **UG906** | Vivado Design Suite Tutorial: Design Flows |
| **UG909** | Vivado Design Suite User Guide: Design Analysis |

## D. 명령어 도움말 보는 법

```tcl
# Tcl Console에서 실시간 도움말
help synth_design
help -args synth_design       ;# 인수 포함 상세 도움말
synth_design -help            ;# 모든 옵션 출력

# 특정 단어가 포함된 명령어 찾기
help -pattern get_*

# 현재 Vivado 버전
puts [version]

# 사용 가능한 모든 명령어
list_cmd
```

## E. 권장 학습 로드맵

```
Step 1: TCL 기초 문법 → [1일]
  - 변수, 리스트, 제어문, 프로시저
  - Vivado Tcl Console에서 실습

Step 2: 객체 모델 이해 → [2일]
  - get_cells/get_pins/get_nets
  - filter 표현식, -of_objects 연결
  - get_property/set_property

Step 3: 프로젝트 모드 → [1일]
  - create_project → add_files → launch_runs
  - open_run → report_* → close_design

Step 4: 비프로젝트 모드 → [2일]
  - read_* → link_design → synth_design
  - opt_design → place_design → route_design
  - write_checkpoint/write_bitstream

Step 5: XDC 타이밍 제약 → [2일]
  - create_clock, set_input_delay/output_delay
  - set_false_path, set_multicycle_path
  - 타이밍 분석 리포트解读

Step 6: 고급 스크립팅 → [3일]
  - Hook Script 자동화
  - 다중 전략 병렬 실행
  - DCP 분석 자동화
  - HTML 대시보드 생성

Step 7: 실전 프로젝트 → [3일+]
  - 종합 빌드 스크립트 작성
  - CI/CD 통합
  - 사내 TCL 라이브러리 구축
```

---

> **최종 팁**: Vivado TCL의 핵심은 **"GUI에서 할 수 있는 모든 것은 TCL로도 할 수 있다"**는 것.
> 복잡한 작업일수록 TCL 스크립트가 더 빠르고 정확하며 재현 가능하다.
> 스크립트를 버전 관리(Git)에 포함시키고, 팀 내 공유를 통해 지식을 축적하라.
