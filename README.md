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

Zmíněné tabulky a sloupce jsem pak mezi sebou propojila pomocí INNER JOIN nejdříve jako VIEW, abych si výsledek spojování mohla prohlédnout a získala tak přehled o budoucí podobě tabulky. Z vytvořeného VIEW jsem pak už vytvořila finální primární tabulku pokrývající data o mzdách a cenách pro roky 2006 až 2018, která jsou potřebná k zodpovězení výzkumných otázek, konkrétně pak otázek 1) až 4).

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
# **Odpovědi na výzkumné otázky**
## **1)   Rostou v průběhu let mzdy ve všech odvětvích, nebo v některých klesají?** 
**Ne, nikoliv.** V průběhu let mzdy sice v převážné většině odvětví rostou, ale nikoliv ve všech, přičemž právě v několika z nich a v různých letech dokonce klesají. Jmenovitě se pak jedná například o odvětví **Těžby a dobývání**, **Ubytování, stravování a pohostinství** a **Zemědělství, lesnictví a rybářství** v roce **2009**. V případě roku **2010** se pokles týkal **Profesní, vědecké a technické činnosti**, **Veřejné správy a obrany**, či **Vzdělávání**. Několik ze sledovaných odvětví čelilo poklesu mezd i v roce **2011, 2014 až 2016**, ale nejzásadnější byl rok **2013**, kdy mzdy poklesly dokonce v **jedenácti odvětvích**.

K získání odpovědi pak bylo využito kódu níže. Za pomoci *industry_avg* jsem sjednotila průměrné mzdy za každé odvětví a rok a odstranila tak jejich duplicity vzniklé původním spojením tabulek mezd a cen potravin. Poté jsem v *wage_growth* spočítala procentuální meziroční růst mezd pomocí funkce LAG(), která umožňuje porovnat danou hodnotou s tou předešlou. Nakonec jsem pomocí hlavního SELECT vyfiltrovala pouze záporné hodnoty meziročního za účelem toho, abych zjistila, zda existují odvětví a roky s poklesem mezd.


    WITH industry_avg AS (
	    SELECT
		    industry,
		    year,
		    ROUND(AVG(avg_wage), 2) AS avg_wage
		 FROM t_natalie_vymlatilova_project_SQL_primary_final
		 WHERE avg_wage IS NOT NULL
		 GROUP BY industry, year
	),
	wage_growth AS (
		SELECT
			industry,
			year,
			avg_wage,
			ROUND(((avg_wage - LAG(avg_wage) OVER (PARTITION BY industry ORDER BY year))
				/ LAG(avg_wage) OVER (PARTITION BY industry ORDER BY year)
				) * 100, 2
			) AS wage_growth_percent
		FROM industry_avg
	)
	SELECT *
	FROM wage_growth
	WHERE wage_growth_percent < 0
	ORDER BY industry, year;

## **2)   Kolik je možné si koupit litrů mléka a kilogramů chleba za první a poslední srovnatelné období v dostupných datech cen a mezd?** 
Za vypočítanou průměrnou mzdu **20 754 Kč k roku 2006** by si bylo možné zakoupit buďto skoro **1 288 kilogramů chleba** nebo přes **1 437 litrů mléka**. Za vypočítanou průměrnou mzdu **32 536 Kč k roku 2018** by to pak bylo přes **1342 kilogramů chleba** nebo přes **1 641 litrů mléka**.

K získání odpovědi pak bylo využito kódu níže. Za pomoci *avg_price* jsem si vyfiltrovala data týkající se pouze chleba a mléka a zaokrouhlila je na dvě desetinná místa. Poté jsem v *avg_wage* spočítala souhrnnou průměrnou mzdu všech odvětví pro jednotlivé roky, což bylo nezbytné k plánovanému výpočtu množství chleba a mléka. K němu pak došlo s vytvořeným příkazem *price_wage*, kdy jsem předchozí mezivýpočty spojila a vydělila souhrnnou roční průměrnou mzdu cenou chleba a mléka. S hlavním SELECT jsem už jen seřadila výsledek podle roku.

    WITH avg_price AS (
	    SELECT
		    year,
		    category,
		    ROUND(AVG(avg_price)::numeric, 2) AS avg_price
		FROM t_natalie_vymlatilova_project_SQL_primary_final
		WHERE category IN ('Mléko polotučné pasterované', 'Chléb konzumní kmínový')
		GROUP BY year, category
	),
	avg_wage AS (
		SELECT
			year,
			ROUND(AVG(avg_wage)::numeric, 2) AS avg_wage
		FROM t_natalie_vymlatilova_project_SQL_primary_final
		WHERE avg_wage IS NOT NULL
		GROUP BY year
	),
	price_wage AS (
		SELECT
			p.year,
			p.category,
			p.avg_price,
			w.avg_wage,
			ROUND((w.avg_wage / p.avg_price)::numeric, 2) AS quantity
		FROM avg_price p
		JOIN avg_wage w USING (year)
	)
	SELECT *
	FROM price_wage
	ORDER BY YEAR;

## 3) Která kategorie potravin zdražuje nejpomaleji (je u ní nejnižší percentuální meziroční nárůst)?
Nejpomaleji zdražuje, tj. má nejnižší procentuální meziroční nárůst, kategorie **Banány žluté**. Její průměrný **nárůst ceny odpovídá 0,81 %** za celé sledované období. Čistě pro zajímavost lze zmínit, že se ve výsledné tabulce **nenacházela jako první, nýbrž jako třetí** po kategorii **Cukr krystalový a Rajská jablka červená kulatá**. Zmíněné dvě kategorie však mají záporný průměrný procentuální nárůst, což znamená, že tyto kategorie nikterak nezdražily, ale naopak **zlevnily**.

K získání odpovědi pak bylo využito kódu níže. Za pomoci *avg_prices* jsem si data zaokrouhlila na dvě desetinná místa. Poté jsem v *growth_prices* spočítala procentuální meziroční růst cen jednotlivých kategorií pomocí funkce LAG(), která umožňuje porovnat danou hodnotou s tou předešlou. Nakonec jsem pomocí hlavního SELECT vypočítala průměrný meziroční růst pro každou kategorii za celé sledované období a podle něj i výsledky seřadila.

    WITH avg_prices AS (
	    SELECT
		    year,
		    category,
		    ROUND(AVG(avg_price)::numeric, 2) AS avg_price
	    FROM t_natalie_vymlatilova_project_SQL_primary_final
	    WHERE avg_price IS NOT NULL
	    GROUP BY year, category
	),
	growth_prices AS (
		SELECT
			category,
			year,
			avg_price,
			ROUND(((avg_price - LAG(avg_price) OVER (PARTITION BY category ORDER BY year))
			/ LAG(avg_price) OVER (PARTITION BY category ORDER BY year)
			) * 100::numeric, 2) AS growth_percent
		FROM avg_prices
	)
	SELECT
		category,
		ROUND(AVG(growth_percent)::numeric, 2) AS avg_growth_percent
	FROM growth_prices
	GROUP BY category
	ORDER BY avg_growth_percent;
