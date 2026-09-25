# MEFS® 검사자 자격 온라인 시험

파일 하나(`index.html`)로 동작하는 정적 웹앱입니다. 서버·빌드·설치가 필요 없습니다.

## 구성

| 시험 | 문항 수 | 통과 기준 |
|---|---|---|
| 모듈 1 퀴즈 · 실행기능과 EFgo™의 이해 | 5 | 80% |
| 모듈 2 퀴즈 · MEFS® 검사의 개요 | 7 | 80% |
| 모듈 3 퀴즈 · MEFS® 소프트웨어의 구조 | 7 | 80% |
| 모듈 4 퀴즈 · MEFS® 시행 가이드 | 6 | 80% |
| 최종 인증 시험 · EFgo™ Certification | 11 | 80% |

## 동작 방식

- 한 문항씩 제시되며 **이전 문항으로 돌아갈 수 없습니다.**
- 재응시할 때마다 **문항 순서와 선택지 순서가 무작위로 바뀝니다.** (참/거짓, '모든 선택지가 정답' 문항은 순서 고정)
- 채점 결과는 **점수(%)와 합격 여부만** 표시되고 정답은 공개되지 않습니다.
- 80% 미만이면 **24시간이 지나야 재응시할 수 있습니다.** 대기 중에는 남은 시간이 실시간으로 표시됩니다.
- 정답은 소스에 평문으로 들어 있지 않습니다. SHA-256 해시로만 저장되며, 응시자의 답을 해시로 바꿔 대조합니다. 개발자 도구로 소스를 열어도 정답을 읽을 수 없습니다.
- 통과 여부와 함께 `M2-260724-A3F91C-P` 형태의 **결과 코드**가 발급됩니다. 응시자 ID·모듈·점수·날짜를 조합한 값이라 임의로 만들어 낼 수 없습니다. 응시자가 담당자에게 전달하면 확인용으로 쓸 수 있습니다.

## 배포 방법 (셋 중 하나)

**1. GitHub Pages — 가장 간단, 무료**
1. GitHub에서 새 저장소 생성 (예: `mefs-quiz`, Public)
2. `index.html` 업로드 후 커밋
3. Settings → Pages → Source: `main` / `root` → Save
4. 1~2분 뒤 `https://아이디.github.io/mefs-quiz/` 로 접속

**2. Vercel — 커스텀 도메인 연결이 쉬움**
1. [vercel.com](https://vercel.com) 가입 → New Project
2. `index.html`이 든 폴더를 드래그 앤 드롭 (또는 GitHub 저장소 연결)
3. Framework Preset은 "Other", 빌드 설정 없이 Deploy

**3. Netlify Drop — 가입 없이 즉시**
[app.netlify.com/drop](https://app.netlify.com/drop) 에 폴더를 끌어다 놓으면 바로 주소가 생성됩니다.

> Streamlit은 파이썬 서버가 필요하고 무료 플랜은 슬립·재시작이 잦아 이 용도에는 맞지 않습니다. 정적 배포가 더 빠르고 안정적입니다.

## 응시 결과를 담당자에게 자동 전송하기 (필수 설정)

응시자가 시험을 마치면 응시자 ID·이름·점수·합격 여부가 Google Sheets에 자동으로 한 줄씩 쌓입니다. 합격뿐 아니라 **불합격과 재응시 기록도 모두 남습니다.**

### 1. Google Sheet 준비
새 Google Sheet를 만듭니다. 이름은 자유롭게 (예: `MEFS 검사자 시험 응시기록`).

### 2. Apps Script 붙여넣기
확장 프로그램 → Apps Script → 기본 코드를 지우고 아래를 붙여넣은 뒤 저장합니다.

```javascript
var COOLDOWN_HOURS = 24;   // 불합격 후 재응시 대기 시간

function doGet(e) {
  var p = e.parameter;
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  ensureHeader(sheet);

  var out;
  if (p.action === 'check') {
    out = check(sheet, p.uid, p.module);
  } else {
    sheet.appendRow([new Date(), p.uid, p.name, p.tail, p.moduleName, p.module,
                     Number(p.correct), Number(p.total), Number(p.pct), p.pass, p.code]);
    out = { ok: true };
  }

  return ContentService
    .createTextOutput(p.callback + '(' + JSON.stringify(out) + ')')
    .setMimeType(ContentService.MimeType.JAVASCRIPT);
}

function ensureHeader(sheet) {
  if (sheet.getLastRow() > 0) return;
  sheet.appendRow(['응시 시각', '응시자 ID', '이름', '연락처 끝4자리', '시험',
                   '시험 코드', '정답 수', '총 문항', '점수(%)', '결과', '결과 코드']);
  sheet.getRange(1, 1, 1, 11).setFontWeight('bold').setBackground('#E7F6F3');
  sheet.setFrozenRows(1);
}

// 해당 응시자의 가장 최근 응시 기록을 찾아 대기 여부를 판정합니다.
function check(sheet, uid, mod) {
  var last = sheet.getLastRow();
  if (last < 2) return { ok: true, blockedUntil: 0 };

  var rows = sheet.getRange(2, 1, last - 1, 11).getValues();
  for (var i = rows.length - 1; i >= 0; i--) {
    if (rows[i][1] !== uid || rows[i][5] !== mod) continue;
    if (rows[i][9] === '합격') return { ok: true, blockedUntil: 0, passed: true };
    var until = new Date(rows[i][0]).getTime() + COOLDOWN_HOURS * 3600000;
    return { ok: true, blockedUntil: until > Date.now() ? until : 0 };
  }
  return { ok: true, blockedUntil: 0 };
}
```

### 3. 웹 앱으로 배포
배포 → 새 배포 → 유형 선택(톱니바퀴) → **웹 앱**

- 실행 계정: **나**
- 액세스 권한: **모든 사용자**  ← 이 설정이 아니면 응시자 기록이 전송되지 않습니다.

배포를 누르면 권한 승인 창이 뜹니다. "고급" → "안전하지 않은 페이지로 이동"을 눌러 승인하세요. 본인이 만든 스크립트이므로 정상입니다.

### 4. 주소를 앱에 넣기
발급된 웹 앱 URL(`https://script.google.com/macros/s/.../exec`)을 복사해 `index.html` 상단의 이 줄에 붙여넣습니다.

```javascript
const RECORD_URL = "";   // ← 여기에 웹앱 주소를 붙여넣기
```

수정한 `index.html`을 다시 업로드하면 끝입니다.

### 확인 방법
응시자 화면 하단에 전송 상태가 표시됩니다.

- 초록 점 · "응시 기록이 담당자에게 전송되었습니다." → Sheet에 기록 완료
- 빨간 점 · "기록 전송에 실패했습니다." → 응시자가 결과 코드를 담당자에게 직접 전달, '다시 전송' 버튼으로 재시도 가능

### 응시자 ID

기관 수가 계속 늘어나는 것을 감안해 소속 기관은 입력받지 않고, **이름 + 휴대폰 뒤 4자리**를 응시자 ID로 씁니다.

```
홍길동-1234
```

입력하는 즉시 화면에 ID가 표시되므로 응시자가 직접 확인할 수 있습니다. 재응시 판정도 이 ID를 기준으로 하니, **모든 시험에서 같은 값을 입력하도록 안내**해 주세요. 시트에서는 이 열로 정렬·필터하면 한 사람의 응시 이력이 한눈에 보입니다.

기관 정보가 필요하면 시트에 열을 추가해 담당자가 직접 채우거나, VLOOKUP으로 기존 평가자 자격관리 시트와 연결하는 편이 관리하기 쉽습니다.

### 재응시 24시간 제한

불합격하면 24시간 뒤부터 재응시할 수 있습니다. 대기 중에 해당 시험을 다시 누르면 남은 시간과 응시 가능 시각이 표시됩니다.

판정 순서는 이렇습니다.

1. **서버 기준(주)** — Apps Script가 시트에서 해당 ID의 가장 최근 기록을 찾아 판정합니다. 기기나 브라우저를 바꿔도 적용됩니다.
2. **브라우저 기준(보조)** — 네트워크 장애 등으로 서버 확인이 안 되면 브라우저에 남은 기록으로 판정합니다.
3. 마지막 기록이 **합격**이면 대기 없이 다시 응시할 수 있습니다.

대기 시간을 바꾸려면 두 곳을 같은 값으로 맞추세요.

- `index.html` → `const COOLDOWN_H = 24;`
- Apps Script → `var COOLDOWN_HOURS = 24;`

> 완벽한 차단은 아닙니다. ID를 다르게 입력하면 우회할 수 있습니다. 다만 응시 기록이 시트에 전부 남으므로, 담당자가 명단과 대조하면 바로 드러납니다. 자격 관리 목적에는 이 정도가 실용적입니다.

### 결과 코드가 필요한 이유
결과 코드는 응시자 ID·모듈·점수·날짜를 해시한 값입니다. 응시자가 점수를 임의로 지어내 보고할 수 없고, 전송이 실패했을 때 담당자가 대조할 수 있는 백업 수단입니다. 코드 끝의 `P`는 합격, `R`은 재응시 필요를 뜻합니다.

## 문항 수정하기

`index.html` 안의 `const DATA = [...]` 를 직접 고치면 되지만, 정답 해시를 다시 계산해야 합니다. 문항을 바꿔야 할 때는 원본 빌드 스크립트(`build.py`)로 다시 생성하는 편이 안전합니다.

---

MEFS® and EFgo™ are trademarks of Reflection Sciences, Inc. · © 2026 Playtag Inc. × Reflection Sciences, Inc.
