# Claude Code Custom Skills on Windows
### 강의를 따라 하다 Windows에서 막힌 것들을 해결한 기록

---

## 배경

Claude Code 커스텀 스킬 관련 강의를 따라 하던 중, Windows 환경에서는 그대로 되지 않는 부분들이 있었습니다.  
막히는 부분을 하나씩 해결하다 보니 원인을 이해하게 됐고, 그 과정에서 알게 된 것들을 정리했습니다.

---

## 마주쳤던 문제들

### 문제 1. 스킬이 Claude에 인식되지 않음
강의대로 스킬을 만들었는데 Claude가 인식을 못 했습니다.

**원인**: 강의는 macOS 기준이라 심볼릭 링크가 자동으로 생성되지만, Windows에서는 개발자 모드가 꺼져 있으면 링크 생성이 조용히 차단됩니다. 파일은 만들어졌지만 연결이 안 된 상태였습니다.

**해결**: 개발자 모드 활성화 + `cmd /c "mklink /D"` 방식 사용 (PowerShell의 `New-Item SymbolicLink`는 관리자 권한 필요)

### 문제 2. `/plugin` 실행 시 경로 탐색 에러
스킬 파일을 안전한 곳으로 옮겼더니 경로를 못 찾는 에러가 났습니다.

**원인**: `marketplace.json`에 이전 경로가 그대로 남아 있었습니다.

**해결**: `marketplace.json`에서 해당 스킬 항목 2줄 삭제

### 추가로 알게 된 것
강의에서는 스킬을 플러그인 폴더 안에 그대로 두는데, 플러그인이 업데이트되면 그 안의 파일이 덮어씌워질 수 있습니다.  
AI와 대화하며 다듬어 만든 스킬은 날아가면 똑같이 재현하기 어렵기 때문에, 플러그인과 분리된 독립 경로(`~/.claude/skills/`)에 보관하는 것이 안전합니다.

---

## 최종 구조

```
C:\Users\{user}\.claude\skills\     ← 커스텀 스킬 보관 경로
├── my-skill-creator\SKILL.md       ← 안전 경로에 저장하도록 규칙을 박은 스킬 생성기
├── perf-analyzer\SKILL.md
└── project-ideator\SKILL.md

{project}\.agent\skills\
└── global → ~/.claude/skills       ← ag-skills 명령어로 생성하는 심볼릭 링크
```

---

## `ag-skills` 명령어

새 프로젝트에서 스킬 연동을 자동화하는 커맨드입니다.  
프로젝트 루트에서 한 번 실행하면 심볼릭 링크가 생성되어 커스텀 스킬이 즉시 연동됩니다.

### 최초 1회 설치 (PowerShell)

```powershell
$scriptDir = "$env:USERPROFILE\scripts"
New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null

$script = @'
New-Item -ItemType Directory -Force -Path ".agent\skills" | Out-Null
$target = ".agent\skills\global"
if (Test-Path $target) { Remove-Item $target -Force -Recurse }
$source = "$env:USERPROFILE\.claude\skills"
if (Test-Path $source) {
    cmd /c "mklink /D `"$target`" `"$source`"" | Out-Null
    Write-Host "스킬 연동 완료!"
} else {
    Write-Host "주의: $source 폴더가 없습니다."
}
'@

$script | Out-File -FilePath "$scriptDir\ag-skills.ps1" -Encoding UTF8

$wrapper = "@echo off`npowershell -NoProfile -ExecutionPolicy Bypass -File `"$scriptDir\ag-skills.ps1`""
$wrapper | Out-File -FilePath "$scriptDir\ag-skills.cmd" -Encoding ASCII

$currentPath = [Environment]::GetEnvironmentVariable('PATH', 'User')
if ($currentPath -notlike "*$scriptDir*") {
    [Environment]::SetEnvironmentVariable('PATH', "$scriptDir;$currentPath", 'User')
}

Write-Host "설치 완료! 터미널을 재시작한 뒤 ag-skills 명령어를 사용하세요."
```

### 프로젝트마다 사용

```bash
ag-skills
```

---

## 사전 조건

- Windows 11
- **개발자 모드 활성화**: 설정 → 시스템 → 개발자용 → 개발자 모드 켬
- Claude Code CLI 설치

---

## 참고 문서

- [`docs/claude_skill_troubleshooting.md`](docs/claude_skill_troubleshooting.md) — 에러 원인 분석 및 해결 과정 상세 기록  
- [`docs/terminal_guide_and_skill_script.md`](docs/terminal_guide_and_skill_script.md) — PowerShell vs CMD 가이드 및 ag-skills 설치 스크립트
