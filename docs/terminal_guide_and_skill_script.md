# 윈도우 터미널 가이드 및 Claude 스킬 연동 스크립트

## 1. PowerShell vs CMD 사용 기준 비교

| 구분 | ⬛ CMD (명령 프롬프트) | 🟦 PowerShell |
| :--- | :--- | :--- |
| **핵심 목적** | **단순 실행 및 확인용** | **복잡한 작업 및 자동화 스크립트용** |
| **언제 쓸까?** | 1. 패키지나 모듈 설치할 때<br>2. 파이썬, 노드 등 단순 실행할 때<br>3. 깃(Git) 명령어 칠 때<br>4. 남이 만든 예전 튜토리얼 따라 할 때 | 1. 변수, 조건문(if), 반복문(for)이 필요할 때<br>2. 시스템 설정(레지스트리, 환경변수)을 건드릴 때<br>3. 결과를 가공해서 다른 명령어로 넘겨야 할 때 |
| **대표적인 예시** | `npm install`<br>`python main.py`<br>`git commit -m "..."`<br>`ping google.com` | `$var = "hello"`<br>`cmd /c "mklink /D ..."`<br>`Invoke-RestMethod` (API 호출) |
| **장점** | 가볍고 빠르다.<br>보안 에러가 거의 없어 진입장벽이 낮다. | 코딩(프로그래밍)하듯 터미널을 다룰 수 있다.<br>결과를 텍스트가 아닌 '객체(Object)'로 다룬다. |
| **단점** | 복잡한 윈도우 시스템 제어나 <br>프로그래밍적 논리 구조를 짜기 매우 힘들다. | 기본 보안 정책 때문에 처음 스크립트(.ps1)를 <br>실행할 때 에러가 나기 쉽다. 무겁다. |

---

## 2. 실전 추천 가이드

### 📌 평소 개발할 때는 👉 `CMD` (또는 Git Bash)
VS Code나 Antigravity에서 단순히 코드를 짜고, 서버를 띄우고(`npm run dev`), 코드를 깃허브에 올리는(`git push`) 등 일상적인 개발 작업을 할 때는 가장 가볍고 에러가 안 나는 CMD를 기본 터미널로 설정해 두고 쓰시는 것이 속 편합니다.

### 📌 뭔가 '프로그램' 같은 스크립트를 짜야 할 때는 👉 `PowerShell`
"특정 폴더가 있는지 검사해서, 없으면 만들고, 텍스트 파일을 새로 써서 환경 변수에 등록해라" 와 같은 **자동화 프로그램**을 짤 때는 고민하지 말고 PowerShell을 여시면 됩니다.

---

## 3. Claude Code 스킬 연동을 위한 윈도우용 스크립트

> **윈도우에서 심볼릭 링크를 사용하려면 개발자 모드가 활성화되어 있어야 합니다.**
>
> 개발자 모드가 켜져 있어도 PowerShell의 `New-Item -ItemType SymbolicLink`는 관리자 권한을 요구하는 경우가 있습니다.
> 이 때는 CMD의 `mklink /D`를 사용하면 개발자 모드 권한만으로 심볼릭 링크가 생성됩니다.

아래 스크립트는 노션 강의 원본에 있던 단순 복사 방식이 아닌, **심볼릭 링크(바로가기)를 생성하여 실시간 동기화가 되도록 맞춤형으로 수정한 PowerShell 스크립트**입니다.

```powershell
# 1. 스크립트 저장 폴더 만들기
$scriptDir = "$env:USERPROFILE\scripts"
New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null

# 2. 심볼릭 링크(바로가기)를 만들어주는 윈도우용 스크립트 작성 (.ps1)
$script = @'
New-Item -ItemType Directory -Force -Path ".agent\skills" | Out-Null
$target = ".agent\skills\global"
if (Test-Path $target) { Remove-Item $target -Force -Recurse }
$source = "$env:USERPROFILE\.claude\skills"
if (Test-Path $source) {
    # PowerShell New-Item은 관리자 권한 요구 → cmd mklink /D 사용 (개발자 모드면 동작)
    cmd /c "mklink /D `"$target`" `"$source`"" | Out-Null
    Write-Host "✅ 스킬 연동 완료! (심볼릭 링크로 연결됨)"
} else {
    Write-Host "⚠️ 주의: $source 폴더가 없습니다."
}
'@

$script | Out-File -FilePath "$scriptDir\ag-skills.ps1" -Encoding UTF8

# 3. 윈도우 특성 보완: 확장자 없이 'ag-skills'만 쳐도 실행되도록 보조 파일(.cmd) 생성
$wrapper = "@echo off`npowershell -NoProfile -ExecutionPolicy Bypass -File `"$scriptDir\ag-skills.ps1`""
$wrapper | Out-File -FilePath "$scriptDir\ag-skills.cmd" -Encoding ASCII

# 4. PATH 환경변수에 등록하여 어디서든 ag-skills 입력 가능하게 만들기
$currentPath = [Environment]::GetEnvironmentVariable('PATH', 'User')
if ($currentPath -notlike "*$scriptDir*") {
    [Environment]::SetEnvironmentVariable('PATH', "$scriptDir;$currentPath", 'User')
}

Write-Host "🎉 설치 끝! 터미널(VS Code 등)을 완전히 껐다 켠 뒤부터 'ag-skills' 명령어를 사용할 수 있습니다."
```

### 💡 실행 방법 안내
1. **최초 1회 필수**: 새로 등록된 환경 변수(PATH)를 인식시키기 위해, **VS Code 등 에디터를 아예 완전히 종료했다가 다시 실행**해 주세요.
2. 이후부터는 프로젝트 루트 디렉토리에서 터미널에 아래 명령어만 치시면 됩니다.
   ```powershell
   ag-skills
   ```
*(참고: 에디터를 재시작하기 번거롭다면, 터미널에 `~\scripts\ag-skills` 라고 전체 경로를 입력하시면 지금 당장 즉시 동작합니다.)*

---

## 4. 커스텀 스킬 관리 방법

### 스킬 저장 위치
커스텀 스킬(직접 만든 스킬)은 반드시 아래 경로에 보관합니다.

```
C:\Users\{사용자명}\.claude\skills\
├── perf-analyzer\
│   └── SKILL.md
└── project-ideator\
    └── SKILL.md
```

- 이 경로는 Claude Code가 자동으로 인식하는 **로컬 커스텀 스킬 전용 공간**입니다.
- 플러그인 폴더(`plugins/`) 안에 커스텀 스킬을 넣으면 업데이트 시 충돌이 발생합니다.

### 커스텀 스킬을 플러그인 폴더에 두면 안 되는 이유

`~\.claude\plugins\marketplaces\` 및 `plugins\cache\` 폴더는 공식 플러그인(`example-skills` 등)이 관리하는 영역입니다. 이 안의 `marketplace.json`에 커스텀 스킬 경로가 등록되어 있어도, 플러그인 캐시(`cache/`)에 실제 파일이 없으면 `/plugin` 명령 실행 시 아래와 같은 에러가 납니다.

```
skills path not found: ...plugins\cache\...\example-skills\...\skills\perf-analyzer
```

**해결 원칙**: 커스텀 스킬은 `~\.claude\skills\`에만 두고, `marketplace.json`에는 등록하지 않는다.
