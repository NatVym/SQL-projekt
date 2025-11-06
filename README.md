# SQL-projekt
# **Popis projektu**

Projekt se zaměřuje na **analýzu dostupnosti základních potravin** pro širokou veřejnost v různých evropských státech, se zvláštním důrazem na situaci v **České republice**.

Součástí projektu je také příprava **doplňkových datových materiálů** obsahujících informace o **HDP**, **GINI koeficientu** a **populaci vybraných evropských států** ve stejném časovém období. Tyto údaje poslouží jako referenční rámec pro lepší pochopení **ekonomického a sociálního kontextu** sledovaných rozdílů.


# **Cíl projektu**
Cílem je **připravit robustní datové podklady** ve **dvou zdrojových datových tabulkách**, které umožní porovnání **dostupnosti** vybraných základních potravin na základě **průměrných příjmů** v definovaných časových obdobích. 

Výsledky **zachycené v primární a sekundární tabulce** budou využity k **zodpovězení pětice výzkumných otázek** týkající se **životní úrovně obyvatel** a budou sloužit jako podklad pro prezentaci na **odborné konferenci**. Zároveň pak zodpovězení otázek bude představovat **naplnění cíle** projektu.

Výstupem tak bude primárně **přehledová analýza dostupnosti základních potravin v České republice**, doplněná o **mezinárodní porovnání**.

# **Výzkumné otázky**
1)   Rostou v průběhu let mzdy ve všech odvětvích, nebo v některých klesají?
    
2)   Kolik je možné si koupit litrů mléka a kilogramů chleba za první a poslední srovnatelné období v dostupných datech cen a mezd?
    
3)   Která kategorie potravin zdražuje nejpomaleji (je u ní nejnižší percentuální meziroční nárůst)?
    
4)   Existuje rok, ve kterém byl meziroční nárůst cen potravin výrazně vyšší než růst mezd (větší než 10 %)?
    
5)  Má výška HDP vliv na změny ve mzdách a cenách potravin? Neboli, pokud HDP vzroste výrazněji v jednom roce, projeví se to na cenách potravin či mzdách ve stejném nebo následujícím roce výraznějším růstem?
# **Vytvoření primární a sekundární tabulky**
## **Primární tabulka** 
Postup ke vzniku primární tabulky byl následující. 

Nejprve jsem udělala průzkum toho, jaké faktové a dimenzní tabulky k zodpovězení otázek potřebuji. Pro zodpovězení čtveřice výzkumných otázek pak k tomu bylo zapotřebí spojení čtyřech tabulek, a to tabulek *czechia_payroll*, *czechia_payroll_industry_branch*, *czechia_price* a *czechia_price_category*. 

Dále jsem pak určila, které sloupce z tabulek jsou ty potřebné, a měla tak v tabulce pouze ty podstatné. Jako takové sloupce byly pro tabulky týkající se mezd zvoleny *cp.payroll_year*, *cpib.name* a *cp.value*, pro tabulky týkající se cen pak *cp.date_from*, *cpc.name*, *cpc.price_unit* a *cp.value*.

Zmíněné tabulky a sloupce jsem pak mezi sebou propojila pomocí INNER JOIN nejdříve jako VIEW, abych si výsledek spojování mohla prohlédnout a získala tak přehled o budoucí podobě tabulky. Z vytvořeného VIEW jsem pak už vytvořila finální primární tabulku pokrývající data o mzdách a cenách pro roky 2006 až 2018, která jsou potřebná k zodpovězení výzkumných otázek.

    CREATE OR REPLACE VIEW v_primary AS 
	WITH wages AS (
		    SELECT
			    cp.payroll_year AS year,
			    cpib.name AS industry,
			    AVG(cp.value) AS avg_wage
			FROM czechia_payroll cp
			INNER JOIN czechia_payroll_industry_branch cpib
			ON cp.industry_branch_code = cpib.code
			WHERE cp.value_type_code = 5958 -- průměrná hrubá mzda
			GROUP BY cp.payroll_year, cpib.name
			),
		prices AS (
			SELECT
				date_part('year', cp.date_from) AS year,
				cpc.name AS category,
				cpc.price_unit,
				AVG(cp.value) AS avg_price
			FROM czechia_price cp
			INNER JOIN czechia_price_category cpc
			ON cp.category_code = cpc.code
			GROUP BY date_part('year', cp.date_from), cpc.name, cpc.price_unit
			)
	SELECT
		p.year,
		w.industry,
		w.avg_wage,
		p.category,
		p.price_unit,
		p.avg_price
	FROM prices p
	LEFT JOIN wages w
	ON p.year = w.year;

    CREATE TABLE t_natalie_vymlatilova_project_SQL_primary_final AS
	    SELECT * 
	    FROM v_primary;


## **Sekundární tabulka** 
