# Local Genomic Database Documentation

이 문서는 `PrimerDesigner` 알고리즘 구동을 위해 구축된 로컬 SQLite 데이터베이스(`annotations.db`)의 상세 명세서입니다.

## 1. 개요 (Overview)
- **Database File**: `database/annotations.db`
- **Format**: SQLite3
- **Size**: Approx. 760 MB (Lightweight Optimized)
- **Target Species**: *Homo sapiens* (Human)
- **Reference Assembly**: **GRCh38 (hg38)**
- **Purpose**: 프라이머 디자인 시 비특이적 결합(Off-target), 변이(SNP), 제한효소 절단 부위 등을 실시간으로 필터링하기 위함.

## 2. 데이터 소스 및 구성 (Data Sources & Schema)

모든 데이터는 프라이머 디자인 속도 최적화를 위해 **위치 정보(Coordinates)** 중심으로 경량화되어 구축되었습니다.

### 데이터 요약 (Summary)

| 테이블명 (Table) | 데이터 유형 (Type) | 레코드 수 (Records) | 컬럼 구성 (Columns) | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| **`exon`** | Genomic Feature | 3,673,949 | id, chrom, start, end, transcript_id | 전사체(Transcript)의 엑손 좌표 정보. Ensembl ID 기반. |
| **`snp`** | Variation | 4,283,874 | id, chrom, pos | 주요 변이(SNP)의 위치 정보. 프라이머 3' 말단 회피용. |
| **`repeats`** | Repeats | 5,683,690 | id, chrom, start, end | 반복 서열(RepeatMasker) 구간 정보. 비특이적 결합 방지용. |
| **`restriction_site`** | Enzyme Sites | 2,091,546 | id, chrom, name, start, end | 주요 제한효소(EcoRI 등) 인식 부위. 클로닝 실험 설계용. |

---

## 3. 상세 스키마 정보 (Detailed Schema)

실제 DB에 적용된 테이블 구조(DDL)입니다.

### 3.1. Exon Table
유전자의 전사체(Transcript) 구조를 정의합니다. 유전자 심볼 대신 고유한 **Transcript ID**를 사용하여 정확도를 높였습니다.
```sql
CREATE TABLE exon (
    id INTEGER PRIMARY KEY,  -- 고유 ID (Auto Increment)
    chrom TEXT,              -- 염색체 번호 (e.g., 'chr1')
    start INTEGER,           -- 엑손 시작 위치
    end INTEGER,             -- 엑손 종료 위치
    transcript_id TEXT       -- Ensembl Transcript ID (e.g., 'ENST00000832824.1')
);
CREATE INDEX idx_exon_loc ON exon(chrom, start, end);

```

### 3.2. SNP Table

프라이머 디자인에 필수적인 **위치 정보(Pos)**만 저장하여 조회 속도를 극대화했습니다. (Ref/Alt 염기 정보 제외)

```sql
CREATE TABLE snp (
    id INTEGER PRIMARY KEY,
    chrom TEXT,              -- 염색체 번호 (e.g., '1')
    pos INTEGER              -- 변이 위치 (1-based coordinate)
);
CREATE INDEX idx_snp_loc ON snp(chrom, pos);

```

### 3.3. Repeats Table

반복 서열의 종류(Class/Family) 구분 없이, 피해야 할 **위험 구간(Masking Region)** 정보만 통합하여 저장했습니다.

```sql
CREATE TABLE repeats (
    id INTEGER PRIMARY KEY,
    chrom TEXT,              -- 염색체 번호 (e.g., 'chr1')
    start INTEGER,           -- 반복 구간 시작
    end INTEGER              -- 반복 구간 종료
);
CREATE INDEX idx_repeats_loc ON repeats(chrom, start, end);

```

### 3.4. Restriction Site Table

Cloning 등 후속 실험에 영향을 줄 수 있는 제한효소 절단 부위입니다. 효소 이름(`name`)을 포함합니다.

```sql
CREATE TABLE restriction_site (
    id INTEGER PRIMARY KEY,
    chrom TEXT,              -- 염색체 번호 (e.g., 'chr1')
    name TEXT,               -- 제한효소 이름 (e.g., 'EcoRI')
    start INTEGER,           -- 인식 부위 시작
    end INTEGER              -- 인식 부위 종료
);
CREATE INDEX idx_rsite_loc ON restriction_site(chrom, start, end);

```

---

## 4. 데이터 업데이트 및 참고 사항

### 주의사항 (Known Issues)

* **Chromosome Notation**: `exon`, `repeats`, `restriction_site` 테이블은 `'chr1'` 형식을 사용하지만, `snp` 테이블은 숫자 `'1'` 형식을 사용할 수 있습니다. 쿼리 작성 시 이를 고려하여 정규화(Normalization)가 필요할 수 있습니다.
* **SNP Filtering**: 현재 DB에는 위치 정보만 존재하므로, 해당 위치에 어떤 변이(A->G 등)가 있는지는 알 수 없으나, 프라이머 디자인 관점에서는 **해당 위치를 피한다**는 목적에 충분합니다.

### 참고 문헌 (References)

1. **Ensembl Genome Browser**: Transcript ID 및 Exon 좌표 원본 데이터.
2. **dbSNP (NCBI)**: Human genetic variation database.
3. **UCSC Genome Browser**: RepeatMasker 및 Restriction Enzyme tracks.