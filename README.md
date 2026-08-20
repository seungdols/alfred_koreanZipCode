# Alfred Juso & English Address Search Workflow (`zip`)

행정안전부 주소정보누리집 데이터를 기반으로 **도로명 한글 주소** 및 **영문 변환 주소**를 실시간으로 검색하고 클립보드에 바로 복사할 수 있는 Alfred 워크플로우입니다.

별도의 **API 키 발급이 필요 없으며**, 외부 의존성 없이 macOS 기본 Python 3 환경에서 가볍고 빠르게 동작합니다.

---

## 📌 주요 기능

- **한글 / 영문 주소 동시 제공**: 한 번의 검색으로 한글 도로명 주소와 영문 표기 주소를 세트로 노출합니다.
- **간편한 클립보드 복사**: 원하는 주소 형식을 선택하고 `Enter`를 누르면 클립보드에 즉시 복사됩니다.
- **No API Key**: 별도 승인키 없이 즉시 실행 가능합니다.
- **가벼운 실행 환경**: 외부 라이브러리 설치 없이 Python 3 표준 모듈만 사용합니다.

---

## 🚀 사용 방법

![image](sample_image.png)

Alfred 검색창을 열고 `zip` 키워드 뒤에 검색할 도로명, 건물명, 지번을 입력합니다.

```text
zip [검색할 주소]

```

### 💡 사용 예시

1. **`zip 판교역로 166`** 입력
2. 결과 목록 확인:
* 🇰🇷 `[13529] 경기도 성남시 분당구 판교역로 166 (백현동)`
* 🇺🇸 `166, Pangyoyeok-ro, Bundang-gu, Seongnam-si, Gyeonggi-do, Republic of Korea, 13529`


3. 원하는 주소를 선택 후 `Enter`를 누르면 클립보드에 복사됩니다.

---

## 🛠 워크플로우 설정 방법

1. **Alfred Preferences** > **Workflows** 이동 후 좌측 하단 `+` > **Blank Workflow** 생성
* **Name**: `Korean & English Address Search`


2. 캔버스 우클릭 > **Inputs** > **Script Filter** 추가
* **Keyword**: `zip`
* **with space**: 체크 (`v`)
* **Argument Required**: 체크 (`v`)
* **Language**: `/bin/zsh`
* **Script**: 아래 파이썬 스크립트 코드 붙여넣기


3. 캔버스 우클릭 > **Outputs** > **Copy to Clipboard** 추가
* **Clipboard Text**: `{query}`
* *(선택)* **Automatically paste to front most app**: 체크 시 복사와 동시에 현재 포커스된 앱에 자동 붙여넣기


4. **Script Filter** 블록의 오른쪽 탭을 드래그하여 **Copy to Clipboard** 블록과 연결합니다.

---

## 📜 Script Filter 코드

```bash
```
```bash
/usr/bin/python3 - << 'EOF'
import sys
import json
import urllib.parse
import urllib.request
import unicodedata

raw_query = """{query}""".strip()
if not raw_query:
    print(json.dumps({"items": []}))
    sys.exit(0)

# 한글 자모 분리 방지 (NFC 정규화)
q_nfc = unicodedata.normalize('NFC', raw_query)

# juso.go.kr 내부 검색 엔드포인트 파라미터 구성
params = urllib.parse.urlencode({
    'keyword': q_nfc,
    'currentPage': '1',
    'countPerPage': '5',
    'resultType': 'json'
}).encode('utf-8')

url = "[https://www.juso.go.kr/addrlink/addrLinkApiJsonp.do](https://www.juso.go.kr/addrlink/addrLinkApiJsonp.do)"

headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Referer': '[https://www.juso.go.kr/openIndex.do](https://www.juso.go.kr/openIndex.do)',
    'Content-Type': 'application/x-www-form-urlencoded'
}

items = []

try:
    req = urllib.request.Request(url, data=params, headers=headers)
    with urllib.request.urlopen(req, timeout=3) as res:
        raw_res = res.read().decode('utf-8').strip()
        
        # JSONP 괄호 제거
        if raw_res.startswith("(") and raw_res.endswith(")"):
            raw_res = raw_res[1:-1]
            
        data = json.loads(raw_res)
        juso_list = data.get('results', {}).get('juso', [])
        
        if juso_list:
            for idx, addr in enumerate(juso_list):
                road_ko = addr.get('roadAddr', '')
                jibun_ko = addr.get('jibunAddr', '')
                eng_addr = addr.get('engAddr', '')
                zip_no = addr.get('zipNo', '')
                
                # 1. 한글 도로명 주소
                ko_val = f"[{zip_no}] {road_ko}" if zip_no else road_ko
                items.append({
                    "uid": f"ko_{idx}",
                    "title": f"🇰🇷 {ko_val}",
                    "subtitle": f"지번: {jibun_ko} (Enter: 한글 주소 복사)",
                    "arg": ko_val,
                    "valid": True
                })
                
                # 2. 영문 도로명 주소
                if eng_addr:
                    en_val = f"{eng_addr}, Republic of Korea, {zip_no}" if zip_no else f"{eng_addr}, Republic of Korea"
                    items.append({
                        "uid": f"en_{idx}",
                        "title": f"🇺🇸 {en_val}",
                        "subtitle": f"한글: {road_ko} (Enter: 영문 주소 복사)",
                        "arg": en_val,
                        "valid": True
                    })

except Exception:
    pass

if not items:
    items.append({
        "title": f"'{raw_query}' 검색 결과 없음",
        "subtitle": "도로명이나 건물명을 올바르게 입력했는지 확인해 주세요.",
        "valid": False
    })

print(json.dumps({"items": items}, ensure_ascii=False))
EOF

```


## 📋 시스템 요구사항

* **macOS** (Python 3 기본 내장)
* **Alfred 4 또는 Alfred 5** (Powerpack 필요)
