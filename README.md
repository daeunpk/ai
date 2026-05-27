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
  images/                 # OCR을 실행할 메뉴판 이미지 보관 폴더
  sample_data/            # Azure 호출 없이 parser를 테스트하는 mock 데이터
  outputs/
    raw/                  # Azure OCR에서 추출한 원본 line JSON 저장
    final/                # 백엔드 팀에 전달할 최종 메뉴 구조화 JSON 저장
    model_compare/        # 모델별 OCR 비교 결과 저장
  ocr_client.py           # Azure Document Intelligence 호출과 raw line 추출
  menu_dictionary.py      # 메뉴명 오타 보정과 카테고리 매칭에 사용하는 음식명 사전
  normalizer.py           # 가격 보정, 메뉴명 정규화, 메뉴명 후보 필터링
  parser.py               # OCR line을 행 단위로 묶고 메뉴명/가격 후보 생성
  preprocess_image.py     # OCR 정확도를 높이기 위한 로컬 이미지 전처리
  reprocess_raw.py        # 저장된 raw OCR JSON을 Azure 재호출 없이 후처리
  main.py                 # 실제 이미지 1장을 OCR 후 최종 JSON 생성
  compare_models.py       # 필요할 때만 Azure OCR 모델별 결과 비교
  test_parser_with_mock.py # mock OCR line JSON으로 parser만 로컬 테스트
  requirements.txt        # Python 패키지 목록
  .env.example            # Azure endpoint/key 작성 예시
  README.md               # 실행 방법과 협업 전달 문서
```

## 협업 흐름

이 모듈은 첫 번째 단계인 OCR/메뉴판 구조화만 담당합니다.

1. 메뉴판 이미지를 `images/`에 넣습니다.
2. `main.py`로 Azure OCR을 실행합니다.
3. OCR raw line은 `outputs/raw/`에 저장됩니다.
4. 메뉴명/가격 후보가 포함된 최종 JSON은 `outputs/final/`에 저장됩니다.
5. 백엔드 팀은 `outputs/final/이미지명_result.json` 파일을 받아 메뉴 매칭과 식단 위험 판단에 사용합니다.

## .env 작성 방법

프로젝트 루트에 `.env` 파일을 만들고 아래 값을 입력합니다.

```bash
AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=https://your-resource-name.cognitiveservices.azure.com/
AZURE_DOCUMENT_INTELLIGENCE_KEY=your_azure_document_intelligence_key
```

Azure Portal에서 Document Intelligence 리소스로 이동한 뒤 `Resource Management > Keys and Endpoint` 메뉴에서 endpoint와 key를 확인할 수 있습니다.

`.env` 파일에는 실제 key가 들어가므로 Git에 올리면 안 됩니다. 이 프로젝트의 `.gitignore`에는 `.env`가 포함되어 있습니다.

## Azure 비용 아끼는 방법

Azure OCR 호출은 비용이 발생하므로 아래 순서로 테스트하는 것을 권장합니다.

1. parser 수정 후에는 먼저 mock 테스트만 실행합니다.

```bash
python test_parser_with_mock.py --input sample_data/mock_ocr_lines.json
```

2. 실제 Azure 호출은 대표 이미지 몇 장으로만 실행합니다.

```bash
python main.py --image images/menu_001.jpg --model prebuilt-layout
```

3. 모델 비교는 선택 기능입니다. 여러 모델을 호출하므로 꼭 필요할 때만 실행합니다.

```bash
python compare_models.py --image images/menu_001.jpg
```

4. 한 번 생성된 `outputs/raw/` 결과는 보관해두고, parser 실험에는 mock 또는 저장된 raw JSON을 재사용합니다.

저장된 raw OCR 결과로 parser만 다시 실행하려면 아래 명령을 사용합니다. Azure를 다시 호출하지 않습니다.

```bash
python reprocess_raw.py --input outputs/raw/menu_001_prebuilt-layout_raw.json
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
python preprocess_image.py --image images/menu_001.jpg --crop 90,120,1030,590 --scale 2
python main.py --image images/preprocessed/menu_001_preprocessed.jpg --model prebuilt-layout
```

`--crop` 값은 `left,top,right,bottom` 순서입니다. 전처리는 로컬에서만 수행되므로 Azure 비용이 발생하지 않습니다.

## 이미지 OCR 실행

기본 모델은 `prebuilt-layout`입니다.

```bash
python main.py --image images/menu_001.jpg --model prebuilt-layout
```

기본 실행에서는 Azure 호출 전에 로컬 전처리를 자동으로 수행합니다. 전처리 이미지는 `images/preprocessed/`에 저장되고, Azure에는 전처리된 이미지가 전달됩니다. 최종 JSON의 `image`는 원본 이미지명, `ocrImage`는 실제 OCR에 사용한 이미지 경로입니다.

원본 이미지를 그대로 Azure에 보내고 싶으면 아래처럼 실행합니다.

```bash
python main.py --image images/menu_001.jpg --model prebuilt-layout --no-preprocess
```

비용을 아끼기 위해 기본 실행은 `prebuilt-layout`을 먼저 1번만 호출합니다. `prebuilt-layout` 호출이 실패하거나 OCR line이 0개일 때만 `prebuilt-read`로 한 번 더 재시도합니다. 메뉴 후보가 0개라는 이유만으로는 자동 재호출하지 않습니다.

재시도 없이 layout만 실행하고 싶으면 아래처럼 실행합니다.

```bash
python main.py --image images/menu_001.jpg --model prebuilt-layout --no-fallback-read
```

저장 파일:

- OCR raw line: `outputs/raw/menu_001_prebuilt-layout_raw.json`
- 최종 JSON: `outputs/final/menu_001_result.json`

## 모델 비교 실행

일반 실행에서는 모델 비교를 하지 않습니다. `main.py`는 기본적으로 `prebuilt-layout`을 먼저 1번만 호출하고, 실패하거나 OCR line이 0개일 때만 `prebuilt-read`로 한 번 재시도합니다.

아래 명령은 비교 실험이 필요할 때만 사용합니다. 같은 이미지에 대해 `prebuilt-read`, `prebuilt-layout`, `prebuilt-document`를 각각 시도하고 line 개수와 추출 텍스트를 출력합니다.

```bash
python compare_models.py --image images/menu_001.jpg
```

저장 파일:

- `outputs/model_compare/menu_001/prebuilt-read.json`
- `outputs/model_compare/menu_001/prebuilt-layout.json`
- `outputs/model_compare/menu_001/prebuilt-document.json`

참고: Azure Document Intelligence v4에서는 general document 기능이 layout 모델 쪽으로 통합되어 있습니다. 코드에서는 `prebuilt-document` 테스트 시 호환을 위해 API version `2023-07-31`을 사용합니다. 리소스/지역/계정 설정에 따라 `prebuilt-document`가 지원되지 않으면 `.error.json` 파일에 오류를 저장합니다.

## Mock Parser 테스트

Azure 호출 비용 없이 parser만 테스트할 수 있습니다.

```bash
python test_parser_with_mock.py --input sample_data/mock_ocr_lines.json
```

저장 파일:

- `outputs/final/mock_ocr_lines_result.json`

## Spring Boot 백엔드 전달 형식

백엔드 팀에는 `outputs/final/이미지명_result.json` 파일을 전달합니다.

```json
{
  "image": "menu_001.jpg",
  "modelId": "prebuilt-layout",
  "menus": [
    {
      "rawName": "수육국밥",
      "normalizedCandidate": "수육국밥",
      "price": 10000,
      "priceRaw": "10,000",
      "priceCorrected": false,
      "priceWarnings": [],
      "options": [],
      "matchedMenu": null,
      "category": null,
      "matchScore": null,
      "nameCorrected": false,
      "description": "",
      "descriptionLines": [],
      "confidence": 0.88,
      "source": {
        "page": 1,
        "lineTexts": ["수육국밥", "10,000"],
        "bbox": [100.0, 120.0, 580.0, 140.0]
      }
    }
  ],
  "rawLines": [
    {
      "text": "수육국밥",
      "page": 1,
      "x1": 100.0,
      "y1": 120.0,
      "x2": 220.0,
      "y2": 140.0,
      "confidence": 1.0
    }
  ]
}
```

`menus`는 후속 메뉴 매칭과 식단 위험 판단에 사용하고, `rawLines`는 OCR 디버깅과 재분석을 위해 함께 보관합니다.

주요 필드:

- `rawName`: OCR 결과에서 추출한 원본 메뉴명 후보
- `normalizedCandidate`: 공백/오타를 보정한 메뉴명 후보
- `price`: 정수형 가격
- `priceRaw`: OCR에서 읽은 가격 원문
- `priceCorrected`: `8,O00`, `7.000`, `₩9,000`처럼 가격 문자열 보정이 있었는지 여부
- `priceWarnings`: `possible_missing_digit`처럼 사람이 확인해야 할 가격 경고
- `options`: `대짜`, `중짜`처럼 한 메뉴에 여러 가격이 있을 때의 옵션별 가격
- `matchedMenu`: `menu_dictionary.py`의 음식명 사전과 매칭된 표준 메뉴명
- `category`: 사전 매칭 시 연결된 음식 카테고리
- `description`: 메뉴 아래/근처 설명 문장을 묶은 텍스트
- `source`: 이 메뉴 후보가 어떤 OCR line과 좌표에서 왔는지 확인하기 위한 정보
