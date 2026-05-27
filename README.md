# Menu OCR AI

한국 로컬 식당 메뉴판 이미지를 Azure Document Intelligence로 OCR 처리한 뒤, 메뉴명과 가격 후보를 표준 JSON으로 구조화하는 Python MVP 모듈입니다.

이 파트는 OCR과 메뉴판 구조화만 담당합니다. GPT, Azure AI Search, 별도 서버 실행은 사용하지 않습니다.

## 설치 방법

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## 폴더 구조와 역할

```text
menu-ocr-ai/
  ocr_pipeline/           # OCR 실행 코드와 메뉴 JSON 구조화 코드
    main.py               # 실제 이미지 1장을 OCR 후 최종 JSON 생성
    ocr_client.py         # Azure Document Intelligence 호출과 raw line 추출
    parser.py             # OCR line을 행 단위로 묶고 메뉴명/가격 후보 생성
    normalizer.py         # 가격 보정, 메뉴명 정규화, 메뉴명 후보 필터링
    menu_dictionary.py    # 메뉴명 오타 보정과 카테고리 매칭에 사용하는 음식명 사전
    preprocess_image.py   # OCR 정확도를 높이기 위한 로컬 이미지 전처리
    reprocess_raw.py      # 저장된 raw OCR JSON을 Azure 재호출 없이 후처리
    compare_models.py     # 필요할 때만 Azure OCR 모델별 결과 비교
    result_builder.py     # ERD에 맞춘 최종 JSON 구조 생성
    test_parser_with_mock.py # mock OCR line JSON으로 parser만 로컬 테스트
  images/                 # OCR을 실행할 메뉴판 이미지 보관 폴더
  sample_data/            # Azure 호출 없이 parser를 테스트하는 mock 데이터
  outputs/
    raw/                  # Azure OCR에서 추출한 원본 line JSON 저장
    final/                # 백엔드 팀에 전달할 최종 메뉴 구조화 JSON 저장
    model_compare/        # 모델별 OCR 비교 결과 저장
  requirements.txt        # Python 패키지 목록
  .env.example            # Azure endpoint/key 작성 예시
  README.md               # 실행 방법과 협업 전달 문서
```

## 협업 흐름

이 모듈은 첫 번째 단계인 OCR/메뉴판 구조화만 담당합니다.

1. 메뉴판 이미지를 `images/`에 넣습니다.
2. `ocr_pipeline/main.py`로 Azure OCR을 실행합니다.
3. OCR raw line은 `outputs/raw/`에 저장됩니다.
4. 메뉴명 앞뒤의 OCR 잡문자와 용량 표기를 후처리합니다. 예: `■김치찌개–` → `김치찌개`, `■두루치기200g出` → `두루치기`
5. 메뉴명/가격 후보가 포함된 최종 JSON은 `outputs/final/`에 저장됩니다.
6. 백엔드 팀은 `outputs/final/이미지명_result.json` 파일을 받아 메뉴 매칭과 식단 위험 판단에 사용합니다.

## .env 작성 방법

프로젝트 루트에 `.env` 파일이 없으면 새로 만들고 아래 값을 입력합니다.
이미 `.env` 파일이 있으면 다시 만들 필요 없이 기존 파일을 그대로 사용하면 됩니다.

```bash
AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=https://your-resource-name.cognitiveservices.azure.com/
AZURE_DOCUMENT_INTELLIGENCE_KEY=your_azure_document_intelligence_key
```

Azure Portal에서 Document Intelligence 리소스로 이동한 뒤 `Resource Management > Keys and Endpoint` 메뉴에서 endpoint와 key를 확인할 수 있습니다.

`.env` 파일에는 실제 key가 들어가므로 Git에 올리면 안 됩니다. 이 프로젝트의 `.gitignore`에는 `.env`가 포함되어 있습니다.

## 실제 이미지 실행 방법

가장 기본 실행 명령어는 아래와 같습니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_001.jpg --model prebuilt-layout
```

다른 이미지를 실행하려면 `images/` 폴더에 이미지를 넣고 `--image` 값만 바꿉니다.

```bash
python3 ocr_pipeline/main.py --image images/내이미지파일.jpg --model prebuilt-layout
```

실행 전 준비 순서:

1. 프로젝트 루트에 `.env` 파일이 있는지 확인합니다. 없으면 `.env 작성 방법` 섹션대로 만듭니다.
2. 분석할 메뉴판 이미지를 `images/` 폴더에 넣습니다.
3. 위 명령어에서 `--image` 경로를 실제 이미지 파일명으로 바꿔 실행합니다.
4. 실행이 끝나면 `outputs/final/이미지명_result.json` 파일을 확인합니다.

예를 들어 `images/menu_003.jpg`를 실행하면 아래처럼 입력합니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_003.jpg --model prebuilt-layout
```

생성되는 주요 파일:

- `outputs/raw/menu_003_prebuilt-layout_raw.json`: Azure OCR 원본 line 결과
- `outputs/final/menu_003_result.json`: 백엔드와 후속 파트에 전달할 최종 JSON

원본 이미지를 그대로 Azure에 보내고 싶으면 `--no-preprocess` 옵션을 붙입니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_003.jpg --model prebuilt-layout --no-preprocess
```

백엔드 연동을 미리 테스트하려면 이미지 출처, 저장소 키, 이미지 URL을 함께 넘길 수 있습니다.

```bash
python3 ocr_pipeline/main.py \
  --image images/menu_003.jpg \
  --model prebuilt-layout \
  --source camera \
  --storage-key scans/menu_003.jpg \
  --image-url https://example.com/scans/menu_003.jpg
```

이 값들은 최종 JSON의 `menu_image.source`, `menu_image.storage_key`, `menu_image.image_url`에 들어갑니다.

백엔드가 파일을 다시 읽지 않고 실행 결과를 바로 확인하고 싶으면 `--print-json`을 붙입니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_003.jpg --model prebuilt-layout --print-json
```

## Azure 비용 아끼는 방법

Azure OCR 호출은 비용이 발생하므로 아래 순서로 테스트하는 것을 권장합니다.

1. parser 수정 후에는 먼저 mock 테스트만 실행합니다.

```bash
python3 ocr_pipeline/test_parser_with_mock.py --input sample_data/mock_ocr_lines.json
```

2. 실제 Azure 호출은 대표 이미지 몇 장으로만 실행합니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_001.jpg --model prebuilt-layout
```

3. 모델 비교는 선택 기능입니다. 여러 모델을 호출하므로 꼭 필요할 때만 실행합니다.

```bash
python3 ocr_pipeline/compare_models.py --image images/menu_001.jpg
```

4. 한 번 생성된 `outputs/raw/` 결과는 보관해두고, parser 실험에는 mock 또는 저장된 raw JSON을 재사용합니다.

저장된 raw OCR 결과로 parser만 다시 실행하려면 아래 명령을 사용합니다. Azure를 다시 호출하지 않습니다.

```bash
python3 ocr_pipeline/reprocess_raw.py --input outputs/raw/menu_001_prebuilt-layout_raw.json
```

## 이미지 촬영/입력 기준

OCR 정확도는 입력 이미지 품질에 크게 영향을 받습니다. 아래 기준을 지키면 Azure 호출 횟수와 후처리 비용을 줄일 수 있습니다.

- 메뉴판이 화면에 최대한 크게 나오도록 촬영합니다.
- 메뉴판 외의 벽, 포스터, 사람, 테이블 등은 가능하면 적게 포함합니다.
- 글자가 흐리지 않도록 초점을 맞추고 흔들림 없이 촬영합니다.
- 빛 반사, 그림자, 테이프, 가림이 메뉴명과 가격을 덮지 않게 합니다.
- 메뉴판을 비스듬히 찍지 말고 가능한 정면에서 촬영합니다.
- 한 이미지에 여러 메뉴판이나 포스터가 섞이지 않게 합니다.
- 작은 글자가 많은 메뉴판은 더 가까이 찍거나 고해상도 이미지로 입력합니다.
- 메뉴명과 가격이 잘린 이미지는 사용하지 않습니다.
- 종이, 스티커, 반사 등으로 가려진 글자는 OCR로 복구하기 어렵습니다.

권장 입력:

- 메뉴판 1개가 이미지 대부분을 차지하는 사진
- 최소 1280px 이상, 가능하면 2000px 이상 해상도
- JPG 또는 PNG 파일
- 메뉴명과 가격이 모두 눈으로 읽히는 이미지

## OCR이 잘 안 될 때

메뉴판 사진에서 글자가 작거나, 주변 배경이 많이 포함되거나, 반사/가림이 있으면 OCR 품질이 크게 떨어질 수 있습니다. 이 경우 Azure를 여러 번 호출하기 전에 로컬에서 메뉴판 영역만 잘라서 확대/선명화한 이미지를 만든 뒤 그 이미지로 OCR을 실행합니다.

```bash
python3 ocr_pipeline/preprocess_image.py --image images/menu_001.jpg --crop 90,120,1030,590 --scale 2
python3 ocr_pipeline/main.py --image images/preprocessed/menu_001_preprocessed.jpg --model prebuilt-layout
```

`--crop` 값은 `left,top,right,bottom` 순서입니다. 전처리는 로컬에서만 수행되므로 Azure 비용이 발생하지 않습니다.

## 이미지 OCR 실행

기본 모델은 `prebuilt-layout`입니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_001.jpg --model prebuilt-layout
```

기본 실행에서는 Azure 호출 전에 로컬 전처리를 자동으로 수행합니다. 전처리 이미지는 `images/preprocessed/`에 저장되고, Azure에는 전처리된 이미지가 전달됩니다. 최종 JSON은 ERD에 맞춘 `scan_session`, `menu_image`, `menu_analyses` 구조로 저장됩니다.
메뉴명은 최종 JSON 저장 전에 한 번 더 정리되어 앞쪽 기호, 뒤쪽 OCR 잡문자, `200g` 같은 용량 표기가 제거됩니다.

원본 이미지를 그대로 Azure에 보내고 싶으면 아래처럼 실행합니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_001.jpg --model prebuilt-layout --no-preprocess
```

비용을 아끼기 위해 기본 실행은 `prebuilt-layout`을 먼저 1번만 호출합니다. `prebuilt-layout` 호출이 실패하거나 OCR line이 0개일 때만 `prebuilt-read`로 한 번 더 재시도합니다. 메뉴 후보가 0개라는 이유만으로는 자동 재호출하지 않습니다.

재시도 없이 layout만 실행하고 싶으면 아래처럼 실행합니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_001.jpg --model prebuilt-layout --no-fallback-read
```

저장 파일:

- OCR raw line: `outputs/raw/menu_001_prebuilt-layout_raw.json`
- 최종 JSON: `outputs/final/menu_001_result.json`

## 모델 비교 실행

일반 실행에서는 모델 비교를 하지 않습니다. `ocr_pipeline/main.py`는 기본적으로 `prebuilt-layout`을 먼저 1번만 호출하고, 실패하거나 OCR line이 0개일 때만 `prebuilt-read`로 한 번 재시도합니다.

아래 명령은 비교 실험이 필요할 때만 사용합니다. 같은 이미지에 대해 `prebuilt-read`, `prebuilt-layout`, `prebuilt-document`를 각각 시도하고 line 개수와 추출 텍스트를 출력합니다.

```bash
python3 ocr_pipeline/compare_models.py --image images/menu_001.jpg
```

저장 파일:

- `outputs/model_compare/menu_001/prebuilt-read.json`
- `outputs/model_compare/menu_001/prebuilt-layout.json`
- `outputs/model_compare/menu_001/prebuilt-document.json`

참고: Azure Document Intelligence v4에서는 general document 기능이 layout 모델 쪽으로 통합되어 있습니다. 코드에서는 `prebuilt-document` 테스트 시 호환을 위해 API version `2023-07-31`을 사용합니다. 리소스/지역/계정 설정에 따라 `prebuilt-document`가 지원되지 않으면 `.error.json` 파일에 오류를 저장합니다.

## Mock Parser 테스트

Azure 호출 비용 없이 parser만 테스트할 수 있습니다.

```bash
python3 ocr_pipeline/test_parser_with_mock.py --input sample_data/mock_ocr_lines.json
```

저장 파일:

- `outputs/final/mock_ocr_lines_result.json`

## 내가 출력하는 JSON 형식

이 모듈의 최종 출력은 `outputs/final/이미지명_result.json`입니다.
백엔드 팀은 이 파일을 받아 `scan_sessions`, `menu_images`, `menu_analyses`에 저장하거나 후속 분석 파트에 전달합니다.

저장 파일 예시:

- 실제 이미지 OCR: `outputs/final/menu_001_result.json`
- mock parser 테스트: `outputs/final/mock_ocr_lines_result.json`

최종 JSON의 최상위 구조는 아래 3개입니다.

| 키 | 연결되는 ERD 테이블 | 설명 |
| --- | --- | --- |
| `scan_session` | `scan_sessions` | 메뉴판 1회 분석 단위 |
| `menu_image` | `menu_images` | OCR에 사용한 이미지 정보 |
| `menu_analyses` | `menu_analyses` | OCR로 추출한 개별 메뉴 목록 |

후속 분석 영역인 번역, 위험도, 매움 여부, 개별 메뉴 이미지는 이 모듈에서 판단하지 않으므로 `null`로 둡니다.
OCR raw line은 최종 JSON에 포함하지 않고, 디버깅과 재분석을 위해 `outputs/raw/`에 별도 저장합니다.

### JSON 예시

백엔드 팀에는 아래 형식의 JSON을 전달합니다.

```json
{
  "scan_session": {
    "title": "menu_001.jpg",
    "menu_count": 1,
    "risky_menu_count": null,
    "scan_status": "completed",
    "scanned_at": "2026-05-28T12:00:00Z"
  },
  "menu_image": {
    "source": "upload",
    "storage_key": null,
    "image_url": null,
    "mime_type": "image/jpeg",
    "file_size": 123456
  },
  "menu_analyses": [
    {
      "menu_name_ko": "수육국밥",
      "menu_name_en": null,
      "description_ko": "",
      "description_en": null,
      "price_text": "10,000",
      "risk_level": null,
      "is_spicy": null,
      "image_url": null,
      "display_order": 1
    }
  ]
}
```

### 필드 설명

#### `scan_session`

| 필드 | 값 | 설명 |
| --- | --- | --- |
| `title` | 입력 이미지 파일명 | 예: `menu_001.jpg` |
| `menu_count` | 정수 | OCR에서 추출한 메뉴 후보 수 |
| `risky_menu_count` | `null` | 위험 판단 파트에서 채우는 값 |
| `scan_status` | `completed` | OCR과 구조화가 끝난 상태 |
| `scanned_at` | ISO datetime | JSON 생성 시각 |

#### `menu_image`

| 필드 | 값 | 설명 |
| --- | --- | --- |
| `source` | `upload` | 현재는 업로드 이미지 기준 |
| `storage_key` | `null` | 외부 저장소 연동 시 백엔드에서 채우는 값 |
| `image_url` | `null` | 이미지 접근 URL이 생기면 백엔드에서 채우는 값 |
| `mime_type` | MIME 타입 | 입력 이미지 확장자로 추론 |
| `file_size` | 정수 또는 `null` | 로컬 입력 이미지 파일 크기 |

#### `menu_analyses`

| 필드 | 값 | 설명 |
| --- | --- | --- |
| `menu_name_ko` | 문자열 | 사전 매칭 메뉴명, 없으면 정규화된 OCR 메뉴명 |
| `menu_name_en` | `null` | 번역 파트에서 채우는 값 |
| `description_ko` | 문자열 | OCR에서 메뉴 근처 설명을 찾은 경우의 한국어 설명 |
| `description_en` | `null` | 번역 파트에서 채우는 값 |
| `price_text` | 문자열 | OCR에서 읽은 가격 문자열 |
| `risk_level` | `null` | 위험 판단 파트에서 채우는 값 |
| `is_spicy` | `null` | 매움 여부 판단 파트에서 채우는 값 |
| `image_url` | `null` | 개별 메뉴 이미지가 생기면 채우는 값 |
| `display_order` | 정수 | 화면 표시용 메뉴 순서 |

## 백엔드/후속 파트 연결 방법

현재 단계에서는 이 OCR 모듈이 서버를 직접 실행하지 않고, 최종 JSON 파일을 만들어 전달합니다.
연결 방식은 팀 상황에 따라 아래 두 가지 중 하나로 정하면 됩니다.

### 1. 파일 전달 방식

지금 바로 쓰기 가장 단순한 방식입니다.

1. OCR 담당자가 실제 이미지를 실행합니다.
2. `outputs/final/이미지명_result.json` 파일이 생성됩니다.
3. 백엔드가 이 JSON 파일을 읽어서 DB에 저장합니다.
4. 후속 분석 파트가 `menu_analyses` 배열을 받아 번역, 위험도, 매움 여부를 채웁니다.

DB 저장 매핑:

| JSON 키 | 저장 대상 |
| --- | --- |
| `scan_session` | `scan_sessions` |
| `menu_image` | `menu_images` |
| `menu_analyses[]` | `menu_analyses` |

후속 파트가 채우면 되는 필드:

- `menu_name_en`
- `description_en`
- `risk_level`
- `is_spicy`
- `menu_analyses.image_url`
- `scan_session.risky_menu_count`

### 2. API 연결 방식

백엔드와 붙일 때는 `ocr_pipeline/main.py`의 실행 결과 파일을 직접 주고받기보다, 백엔드가 이미지 업로드를 받은 뒤 OCR 모듈을 호출하는 방식이 자연스럽습니다.

추천 흐름:

1. 프론트가 메뉴판 이미지를 백엔드에 업로드합니다.
2. 백엔드는 이미지를 저장하고 OCR 모듈에 이미지 경로를 넘깁니다.
3. OCR 모듈은 `outputs/final/이미지명_result.json`과 같은 구조의 JSON을 생성합니다.
4. 백엔드는 JSON의 `scan_session`, `menu_image`, `menu_analyses`를 DB에 저장합니다.
5. 후속 분석 파트는 저장된 `menu_analyses`를 기준으로 위험도와 번역 필드를 업데이트합니다.

백엔드에서 Python 스크립트를 호출해야 한다면 기본 명령은 아래와 같습니다.

```bash
python3 ocr_pipeline/main.py --image images/menu_003.jpg --model prebuilt-layout
```

이미 백엔드가 이미지를 저장한 뒤 OCR을 호출하는 경우에는 저장 정보를 같이 넘깁니다.

```bash
python3 ocr_pipeline/main.py \
  --image uploads/menu_003.jpg \
  --model prebuilt-layout \
  --source camera \
  --storage-key scans/menu_003.jpg \
  --image-url https://example.com/scans/menu_003.jpg \
  --print-json
```

연동용 옵션:

| 옵션 | JSON 필드 | 설명 |
| --- | --- | --- |
| `--source` | `menu_image.source` | `camera` 또는 `upload` |
| `--storage-key` | `menu_image.storage_key` | S3, 서버 저장소 등에 저장된 이미지 키 |
| `--image-url` | `menu_image.image_url` | 백엔드가 접근 가능한 이미지 URL |
| `--print-json` | stdout | 최종 JSON을 터미널 출력으로도 반환 |

장기적으로는 `ocr_pipeline/result_builder.py`가 만드는 JSON 구조를 유지하면서, 파일 저장 대신 백엔드 API 응답으로 바로 넘기도록 바꾸면 됩니다.
