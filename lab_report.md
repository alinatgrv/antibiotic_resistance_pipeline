# Antibiotic Resistance Project — Variant Calling & Annotation  
**Author:** Алина Тагирова  
**Course:** IB Practicum 2025  

---

# 1. Downloading Reference Genome

Скачиваю референсный геном *E. coli* K-12 MG1655:

wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/005/845/GCF_000005845.2_ASM584v2/GCF_000005845.2_ASM584v2_genomic.fna.gz

wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/005/845/GCF_000005845.2_ASM584v2/GCF_000005845.2_ASM584v2_genomic.gff.gz


Разархивирование:

gunzip GCF_000005845.2_ASM584v2_genomic.fna.gz
gunzip GCF_000005845.2_ASM584v2_genomic.gff.gz


---

# 2. Downloading Sequencing Reads

Скачивание FASTQ ридов:

wget https://figshare.com/ndownloader/files/17826528
 -O resistant_reads_1.fastq.gz
wget https://figshare.com/ndownloader/files/17826531
 -O resistant_reads_2.fastq.gz


Из-за ошибки HTTP скачала вручную через браузер и поместила файлы в `raw_data/`.

Просмотр первых ридов:

gzcat amp_res_1.fastq.gz | head -20
gzcat amp_res_2.fastq.gz | head -20


Подсчёт числа ридов:

gzcat amp_res_1.fastq.gz | wc -l # 1823504
gzcat amp_res_2.fastq.gz | wc -l # 1823504


Каждый рид = 4 строки ->  
**455 876 reads** в каждом файле.

---

# 3. FastQC of Raw Reads

Установка:

mamba install -c bioconda fastqc


Запуск:

fastqc -o fastqc_raw amp_res_1.fastq.gz amp_res_2.fastq.gz


### Основные результаты:

- Total Sequences: **455 876**
- Encoding: **Illumina 1.9 (Phred+33)**
- Красные модули:  
  - Per base sequence quality  
  - Per tile sequence quality  
- Жёлтые:  
  - Per base content  
  - Per sequence GC content  

Качество типичное для сырых ридов, требуется trimming.

---

# 4. Trimming Reads (Trimmomatic)

Установка:

mamba install -c bioconda trimmomatic


Создаю папку:

mkdir -p trimmed


### Команда тримминга:

trimmomatic PE -phred33
amp_res_1.fastq.gz amp_res_2.fastq.gz
trimmed/amp_res_1P.fq.gz trimmed/amp_res_1U.fq.gz
trimmed/amp_res_2P.fq.gz trimmed/amp_res_2U.fq.gz
LEADING:20 TRAILING:20 SLIDINGWINDOW:10:20 MINLEN:20


### Количество ридов после тримминга:

gzcat trimmed/amp_res_1P.fq.gz | wc -l # 1785036
gzcat trimmed/amp_res_2P.fq.gz | wc -l # 1785036


→ **446 259 paired reads**

Потеряно всего **2.1%**, качество сильно улучшилось.

### FastQC after trimming:

fastqc -o fastqc_trimmed trimmed/amp_res_1P.fq.gz trimmed/amp_res_2P.fq.gz


- Резкое улучшение Per-base quality  
- Нет адаптеров  
- Данные готовы для выравнивания  

---

# 5. Alignment with BWA MEM

## 5.1 Installing BWA

mamba install -c bioconda bwa


## 5.2 Indexing reference

bwa index raw_data/GCF_000005845.2_ASM584v2_genomic.fna


BWA создал файлы `.amb`, `.ann`, `.bwt`, `.pac`, `.sa`.

## 5.3 Aligning paired reads

Создаю папку:

mkdir alignment


Выравнивание:

bwa mem raw_data/GCF_000005845.2_ASM584v2_genomic.fna \
  trimmed/amp_res_1P.fq.gz trimmed/amp_res_2P.fq.gz \
  > alignment/alignment.sam


## 5.4 Converting SAM → BAM

samtools view -Sb alignment/alignment.sam > alignment/alignment.bam


## 5.5 Alignment statistics

samtools flagstat alignment/alignment.bam


### Результаты:

- **99.87% mapped reads**
- **99.56% properly paired**
- Singletons: **0.11%**
- Контаминации нет

---

# 6. Sorting & Indexing BAM

samtools sort alignment/alignment.bam -o alignment/alignment_sorted.bam
samtools index alignment/alignment_sorted.bam


---

# 7. Variant Calling (VarScan2)

samtools mpileup -f raw_data/GCF_000005845.2_ASM584v2_genomic.fna
alignment/alignment_sorted.bam |
varscan mpileup2snp --min-var-freq 0.30 --variants --output-vcf 1 \

variants/VarScan_results.vcf


---

# 8. Automatic SNP Annotation (snpEff)

Создаём конфигурацию:

echo "k12.genome : ecoli_K12" > snpEff.config


Скачиваем файл GenBank:

cd raw_data
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/005/845/GCF_000005845.2_ASM584v2/GCF_000005845.2_ASM584v2_genomic.gbff.gz

gunzip GCF_000005845.2_ASM584v2_genomic.gbff.gz
cp GCF_000005845.2_ASM584v2_genomic.gbff ../data/k12/genes.gbk
cd ..


Строим базу:

snpEff build -genbank -v k12


Аннотируем VCF:

snpEff ann k12 variants/VarScan_results.vcf > variants/VarScan_results_annotated.vcf


Аннотированный файл содержит поле **ANN**, включающее:
- gene  
- effect  
- impact  
- amino acid change  
- coding change  

---

# 9. Summary

К получен полный пайплайн:

- Скачана референсная последовательность  
- Проведён QC и trimming  
- Риды выровнены на референс  
- Вызваны SNP  
- Аннотированы при помощи snpEff  


---


# 10.  Таблица аннотированных мутаций (snpEff)

Далее с помощью `awk` извлечена полная таблица аннотаций:

Файл: `variants_table.tsv`
| CHROM | POS | REF | ALT | Gene | Effect | Impact | AA_change |
|-------|------|-----|-----|-------|---------|---------|-----------|
| NC_000913.3 | 93043 | C | G | ftsI | missense_variant | MODERATE | p.Ala544Gly |
| NC_000913.3 | 93043 | C | G | murE | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | murF | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | mraY | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | murD | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | cra | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | mraZ | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | rsmH | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 93043 | C | G | ftsL | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | acrB | missense_variant | MODERATE | p.Gln569Leu |
| NC_000913.3 | 482698 | T | A | pdeB | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | ylaC | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | maa | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | hha | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | tomB | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | acrR | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | mscK | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 482698 | T | A | acrA | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | glnH | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | dps | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | rhtA | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | opgE | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | mntR | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | ybiR | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | ybiT | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | yliM | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | ompX | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | mntS | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | ldtB | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 852762 | A | G | rybA | intragenic_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | mntP | missense_variant | MODERATE | p.Gly25Asp |
| NC_000913.3 | 1905761 | G | A | yoaE | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yoaL | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yobH | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yebQ | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | manX | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | manY | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | manZ | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yobD | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | rlmA | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | cspC | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yobF | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | yebO | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | mgrB | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 1905761 | G | A | kdgR | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | envZ | missense_variant | MODERATE | p.Val241Gly |
| NC_000913.3 | 3535147 | A | C | yhgE | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | greB | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | yhgF | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | hslO | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | pck | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 3535147 | A | C | ompR | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | rsgA | synonymous_variant | LOW | p.Ala252Ala |
| NC_000913.3 | 4390754 | G | T | mscM | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | psd | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | orn | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | yjeV | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | nnr | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | tsaE | upstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | yjeO | downstream_gene_variant | MODIFIER | nan |
| NC_000913.3 | 4390754 | G | T | queG | downstream_gene_variant | MODIFIER | nan |
---


# 11. Results and Discussion

## 11.1 Функциональный анализ значимых мутаций

В результате аннотации выявлены 4 мутации, приводящие к изменениям аминокислотной последовательности:

### **1. ftsI — p.Ala544Gly (missense)**  
Белок: PBP3 (пеннициллин-связывающий белок).  
Функция: биосинтез клеточной стенки.  
Гипотеза: изменение мишени → β-лактамы хуже связываются → устойчивость.

### **2. acrB — p.Gln569Leu (missense)**  
Белок: компонент мульти-лекарственного насоса AcrAB-TolC.  
Функция: выкачивание антибиотиков.  
Гипотеза: активация эффлюксной системы → снижение концентрации антибиотика в клетке.

### **3. envZ — p.Val241Gly (missense)**  
Белок: сенсорная киназа системы EnvZ-OmpR.  
Функция: регуляция поринов OmpF/OmpC.  
Гипотеза: снижение проницаемости наружной мембраны → меньше антибиотика проникает.

### **4. mntP — p.Gly25Asp**  
Белок: Mn²⁺ экспортер.  
Функция: стресс-ответ.  
Гипотеза: вторичная, маловероятно что ключевая.

---

# 11. Предполагаемый механизм устойчивости

На основании выявленных мутаций предполагается комбинация трёх механизмов:

1. **Изменение мишени антибиотика** — *ftsI*  
2. **Повышенная активность эффлюксного насоса** — *acrB*  
3. **Снижение проникновения антибиотика через наружную мембрану** — *envZ*

Эти три мутации вместе объясняют устойчивость к ампициллину.

---

# 12. Рекомендации по лечению

Так как β-лактамы (включая ампициллин) будут неэффективны,  
рекомендуется использовать антибиотики **с другими мишенями**:

### Подходящие варианты:
- **Аминогликозиды** (гентамицин, амикацин)  
- **Тетрациклины** (доксициклин)  
- **Фторхинолоны** (ципрофлоксацин)  
- **Комбинации с ингибиторами β-лактамаз**

### Нежелательно:
- ампициллин  
- амоксициллин  
- цефалоспорины 1 поколения  

---

# 13. Заключение

Проведён полный анализ данных секвенирования: QC, trimming, выравнивание, вызов SNP, аннотация и биологическая интерпретация.  
Ключевыми мутациями, объясняющими устойчивость, являются изменения в генах **ftsI**, **acrB** и **envZ**.  
Выявленные механизмы позволяют предложить корректные варианты терапии.










