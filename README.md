# Analysing World Layoffs using SQL
## Data Cleaning and Transformation

### Create a Duplicate Table
```sql
CREATE TABLE layoffs_staging LIKE layoffs;

INSERT INTO layoffs_staging
SELECT * FROM layoffs;

SELECT * FROM layoffs_staging;
```

### Create a New Table with a Row Number Column
```sql
CREATE TABLE layoffs_staging2 (
    `company` TEXT,
    `location` TEXT,
    `industry` TEXT,
    `total_laid_off` INT DEFAULT NULL,
    `percentage_laid_off` TEXT,
    `date` TEXT,
    `stage` TEXT,
    `country` TEXT,
    `funds_raised_millions` INT DEFAULT NULL,
    `rn` INT -- Add this column for row numbers
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### Insert data with row numbers to identify duplicates
```sql
INSERT INTO layoffs_staging2
SELECT *, 
       ROW_NUMBER() OVER(PARTITION BY company, location, industry, total_laid_off, 
                         percentage_laid_off, `date`, stage, country, 
                         funds_raised_millions) AS rn
FROM layoffs_staging;

DELETE 
FROM layoffs_staging2
WHERE rn > 1;
```
### Standardizing Data

```sql
-- Remove extra spaces from the company column
UPDATE layoffs_staging2
SET company = TRIM(company);

-- Standardize industry names (e.g., 'Crypto' and 'Crypto Currency' -> 'Crypto')
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

-- Fix country names (e.g., 'United States.' -> 'United States')
UPDATE layoffs_staging2
SET country = 'United States'
WHERE country LIKE 'United States%';

-- Convert `date` column from text to date format
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

-- Handle null and missing values in industry column
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';

-- Populate missing industry values using values from other records of the same company
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
   ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL 
  AND t2.industry IS NOT NULL;

-- Delete rows where both `total_laid_off` and `percentage_laid_off` are null
DELETE FROM layoffs_staging2
WHERE total_laid_off IS NULL 
  AND percentage_laid_off IS NULL;

-- Drop the row number column after cleaning
ALTER TABLE layoffs_staging2
DROP COLUMN rn;
``` 
## Exploratory Data Analysis

### 1. Companies that laid off all employees (indicating they went under)
```sql
SELECT * 
FROM layoffs_staging2
WHERE percentage_laid_off = 1
ORDER BY total_laid_off DESC;
```
### 2. Total layoffs by company
```sql
SELECT company, SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY company
ORDER BY total_laid_off DESC;
```

### 3. Total layoffs by industry
```sql
SELECT industry, SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY industry
ORDER BY total_laid_off DESC;
```

### 4. Time range of the dataset
```sql
SELECT MIN(`date`) AS start_date, MAX(`date`) AS end_date
FROM layoffs_staging2;
```

### 5. Total layoffs by country
```sql
SELECT country, SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY country
ORDER BY total_laid_off DESC;
```

### 6. Layoffs by year
```sql
SELECT YEAR(`date`) AS year, SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY YEAR(`date`)
ORDER BY year DESC;
```

### 7. Total layoffs by company stage
```sql
SELECT stage, SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY stage
ORDER BY total_laid_off DESC;
```

### 8. Cumulative sum of layoffs starting from the first month
```sql
WITH cte AS (
    SELECT DATE_FORMAT(`date`, '%Y-%m') AS `month`, 
           SUM(total_laid_off) AS total_off
    FROM layoffs_staging2
    WHERE `date` IS NOT NULL
    GROUP BY `month`
)
SELECT `month`, total_off, 
       SUM(total_off) OVER(ORDER BY `month`) AS cumulative_sum
FROM cte;
```

### 9. Top 5 companies per year based on layoffs
```sql
WITH company_year AS (
    SELECT company, YEAR(`date`) AS year, 
           SUM(total_laid_off) AS total_laid_off
    FROM layoffs_staging2
    GROUP BY company, year
), company_yr_rank AS (
    SELECT *, 
           DENSE_RANK() OVER(PARTITION BY year ORDER BY total_laid_off DESC) AS rankings
    FROM company_year
)
SELECT * 
FROM company_yr_rank
WHERE rankings <= 5;
```

### 10. Average percentage of employees laid off by quarter
```sql
SELECT YEAR(`date`) AS year, QUARTER(`date`) AS quarter, 
       ROUND(AVG(total_laid_off), 2) AS avg_layoff
FROM layoffs_staging2
WHERE `date` IS NOT NULL
GROUP BY year, quarter
ORDER BY year, quarter;
```

### 11. Top 5 companies by total funds raised that have also laid off employees, with layoff percentage
```sql
SELECT company, 
       SUM(funds_raised_millions) AS total_funds, 
       SUM(total_laid_off) AS total_laid_off
FROM layoffs_staging2
GROUP BY company
ORDER BY total_funds DESC
LIMIT 5;
```

### 12. Average percentage of layoffs in each industry, ranked from highest to lowest
```sql
SELECT industry, 
       AVG(total_laid_off) AS avg_laid_off
FROM layoffs_staging2
GROUP BY industry
ORDER BY avg_laid_off DESC;
```

### 13. Year-over-year change in total_laid_off for each industry
```sql
WITH cte AS (
    SELECT industry, YEAR(`date`) AS year, 
           SUM(total_laid_off) AS total_laid_off
    FROM layoffs_staging2
    WHERE industry IS NOT NULL
    GROUP BY industry, year
)
SELECT *, 
       ROUND((total_laid_off - LAG(total_laid_off) OVER(PARTITION BY industry ORDER BY year)) 
             / LAG(total_laid_off) OVER(PARTITION BY industry ORDER BY year) * 100, 2) AS percentage_change
FROM cte
ORDER BY industry, year;
```

### 14. Top three dates with the most layoffs, along with the most affected industries on those dates
```sql
SELECT `date`, 
       SUM(total_laid_off) AS total_laid_off, 
       industry
FROM layoffs_staging2
GROUP BY `date`, industry
ORDER BY total_laid_off DESC
LIMIT 3;
```
