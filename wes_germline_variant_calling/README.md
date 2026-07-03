# WES Germline SNV/Indel Calling

WES(Whole Exome Sequencing) 데이터에서 **생식세포(germline) 단일염기변이(SNV)와 짧은 삽입/결실(Indel)**을 찾아내는
GATK Best Practices 파이프라인을 직접 실행하며 감을 잡기 위한 실습입니다.

## 이 분석이 무엇인가 (개념 정리)

**Germline이란?**
부모로부터 물려받아 몸의 모든 세포에 동일하게 존재하는 선천적 변이입니다.
(반대 개념: Somatic — 암 조직 등에서 후천적으로 생기는 변이. 이번 실습 범위 아님)

**SNV / Indel이란?**
| 용어 | 설명 |
|---|---|
| SNV (Single Nucleotide Variant) | 염기 하나가 다른 염기로 바뀐 변이 (예: A → G) |
| Indel (Insertion/Deletion) | 몇 개의 염기가 짧게 삽입되거나 삭제된 변이 |

(참고로 SNV보다 큰 규모의 구조적 변화 — CNV, SV — 는 이번 실습 범위 밖이며, 별도 분석/툴이 필요합니다.)

**WES에서 왜 "SNV/Indel calling"인가?**
WES는 유전체의 약 1%인 단백질 코딩 영역(exon)만 캡처해서 시퀀싱하기 때문에,
큰 구조 변이(CNV/SV)보다 **점 단위 변이(SNV/Indel) 검출에 최적화**되어 있습니다.
캡처 편향과 낮은 커버리지 균일성 때문에 CNV 검출은 WES만으로는 정확도가 떨어져 별도 접근이 필요합니다.

**전체 그림에서 이 분석의 위치**

|  | SNV/Indel | CNV/SV |
|---|---|---|
| Germline | **← 이번 실습** (HaplotypeCaller) | GATK gCNV, ExomeDepth 등 (별도) |
| Somatic | Mutect2 (별도) | CNVkit 등 (별도) |

## 실습 데이터

**H3ABioNet NEAT 합성 WES 데이터 (chromosome 1, 50X coverage)**
- 출처: [H3ABioNet Variant-Calling SOP](https://h3abionet.github.io/H3ABionet-SOPs/Variant-Calling)
- 다운로드: `http://h3data.cbio.uct.ac.za/assessments/NextGenVariantCalling/practice/`
- NEAT 시뮬레이터로 참조 유전체에 "golden" 변이를 삽입한 뒤 생성한 합성 리드
- 정답 VCF(golden VCF)가 함께 제공되어, 콜한 변이와 직접 비교(concordance 확인) 가능

```bash
mkdir -p data/raw
wget -r -np -nH --cut-dirs=3 -R "index.html*" \
  http://h3data.cbio.uct.ac.za/assessments/NextGenVariantCalling/practice/ \
  -P data/raw
```

## 파이프라인 개요

```
FASTQ
  │  QC                  FastQC / MultiQC
  ▼
Trimmed FASTQ
  │  Alignment            BWA-MEM 정렬 → 좌표 정렬 BAM
  ▼
Sorted BAM
  │  MarkDuplicates        Picard/GATK MarkDuplicates
  ▼
Dedup BAM
  │  BQSR                  BaseRecalibrator → ApplyBQSR
  ▼
Recalibrated BAM
  │  HaplotypeCaller        GATK HaplotypeCaller (GVCF 모드)
  ▼
Raw VCF
  │  Filtering               Hard-filtering (소규모 exome이라 VQSR 대신 hard filter)
  ▼
Filtered VCF  →  golden VCF와 비교(concordance)
```

## 진행 상황

- [ ] 데이터 다운로드 및 확인
- [ ] 환경 세팅 (environment.yml)
- [ ] Step 1: QC
- [ ] Step 2: Alignment (BWA-MEM)
- [ ] Step 3: MarkDuplicates
- [ ] Step 4: BQSR
- [ ] Step 5: HaplotypeCaller
- [ ] Step 6: Filtering
- [ ] Golden VCF와 concordance 비교
- [ ] (선택) SnpEff/ANNOVAR 주석

세부 진행 로그는 [`notes/log.md`](notes/log.md) 에 날짜별로 기록합니다.

## 환경 설정

```bash
conda env create -f environment.yml
conda activate wes_germline_variant_calling
```

## 디렉토리 구조

```
wes_germline_variant_calling/
├── README.md
├── environment.yml
├── scripts/            # 단계별 실행 스크립트 (실습하며 하나씩 추가)
├── notes/
│   └── log.md            # 실습 일지
├── data/                  # (gitignore) 원본 데이터, 로컬에만 존재
│   ├── raw/
│   ├── reference/
│   └── intermediate/
└── results/
    ├── qc_summary/          # MultiQC 리포트 등 (용량 작으면 커밋)
    └── final.filtered.vcf   # 최종 결과 (용량 작아서 커밋 가능)
```

## 참고자료
- [GATK Best Practices Workflow](https://gatk.broadinstitute.org/hc/en-us/sections/360007226651-Best-Practices-Workflows)
- [H3ABioNet Variant-Calling SOP](https://h3abionet.github.io/H3ABionet-SOPs/Variant-Calling)
