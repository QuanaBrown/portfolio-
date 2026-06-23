-- SETUP: Run this once to create your practice schema
CREATE TABLE departments (
  dept_id    NUMBER PRIMARY KEY,
  dept_name  VARCHAR2(50),
  location   VARCHAR2(50),
  budget     NUMBER(12,2)
);

CREATE TABLE employees (
  emp_id      NUMBER PRIMARY KEY,
  first_name  VARCHAR2(50),
  last_name   VARCHAR2(50),
  dept_id     NUMBER REFERENCES departments(dept_id),
  job_title   VARCHAR2(80),
  salary      NUMBER(10,2),
  hire_date   DATE,
  manager_id  NUMBER
);

CREATE TABLE sales_orders (
  order_id    NUMBER PRIMARY KEY,
  emp_id      NUMBER REFERENCES employees(emp_id),
  region      VARCHAR2(30),
  product     VARCHAR2(50),
  amount      NUMBER(10,2),
  order_date  DATE,
  status      VARCHAR2(20)
);

INSERT INTO departments VALUES (1,'Information Technology','Chicago',850000);
INSERT INTO departments VALUES (2,'Finance','Houston',620000);
INSERT INTO departments VALUES (3,'Operations','Dallas',740000);
INSERT INTO departments VALUES (4,'Sales','Charlotte',920000);
INSERT INTO departments VALUES (5,'HR','Detroit',310000);

INSERT INTO employees VALUES (101,'Marcus','Reed',1,'Systems Analyst',95000,DATE '2019-03-12',NULL);
INSERT INTO employees VALUES (102,'Priya','Nair',1,'IT Project Manager',118000,DATE '2017-08-01',101);
INSERT INTO employees VALUES (103,'Jordan','Wells',2,'Financial Analyst',87000,DATE '2020-11-15',NULL);
INSERT INTO employees VALUES (104,'Aisha','Brown',3,'Operations Analyst',92000,DATE '2018-06-22',NULL);
INSERT INTO employees VALUES (105,'Tyler','Moss',4,'Sales Engineer',105000,DATE '2021-01-09',NULL);
INSERT INTO employees VALUES (106,'Carmen','Diaz',4,'Business Analyst',98000,DATE '2016-04-30',105);
INSERT INTO employees VALUES (107,'Kevin','Park',1,'Cloud Architect',135000,DATE '2015-02-17',102);
INSERT INTO employees VALUES (108,'Simone','Grant',5,'HR Analyst',76000,DATE '2022-07-11',NULL);
INSERT INTO employees VALUES (109,'Darius','Osei',2,'Senior Analyst',110000,DATE '2018-09-03',103);
INSERT INTO employees VALUES (110,'Nina','Vasquez',3,'IT Coordinator',82000,DATE '2023-01-20',104);

INSERT INTO sales_orders VALUES (1001,105,'Midwest','Cloud Storage',45200,DATE '2024-01-15','Closed');
INSERT INTO sales_orders VALUES (1002,106,'South','ERP License',88500,DATE '2024-02-03','Closed');
INSERT INTO sales_orders VALUES (1003,105,'Midwest','Security Suite',62000,DATE '2024-02-20','Closed');
INSERT INTO sales_orders VALUES (1004,106,'Southeast','Cloud Storage',31000,DATE '2024-03-08','Open');
INSERT INTO sales_orders VALUES (1005,105,'Midwest','ERP License',95000,DATE '2024-03-22','Closed');
INSERT INTO sales_orders VALUES (1006,109,'South','Analytics Platform',74000,DATE '2024-04-10','Closed');
INSERT INTO sales_orders VALUES (1007,107,'West','Cloud Storage',120000,DATE '2024-04-18','Closed');
INSERT INTO sales_orders VALUES (1008,106,'Southeast','Security Suite',55000,DATE '2024-05-01','Closed');
INSERT INTO sales_orders VALUES (1009,105,'Midwest','Analytics Platform',88000,DATE '2024-05-14','Open');
INSERT INTO sales_orders VALUES (1010,109,'South','ERP License',102000,DATE '2024-06-02','Closed');

COMMIT;
SELECT
  d.dept_name,
  d.location,
  COUNT(e.emp_id)          AS headcount,
  ROUND(AVG(e.salary), 0)  AS avg_salary,
  MAX(e.salary)            AS max_salary,
  SUM(e.salary)            AS total_payroll
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_name, d.location
ORDER BY total_payroll DESC;
SELECT 
    department_id, 
    COUNT(*) AS employee_count, 
    AVG(salary) AS average_salary, 
    MAX(salary) AS max_salary
FROM 
    employees
GROUP BY 
    department_id;
SELECT
  first_name || ' ' || last_name          AS full_name,
  job_title,
  hire_date,
  ROUND(MONTHS_BETWEEN(SYSDATE, hire_date) / 12, 1) AS years_tenure,
  salary,
  CASE
    WHEN MONTHS_BETWEEN(SYSDATE, hire_date) / 12 >= 7 THEN 'Veteran'
    WHEN MONTHS_BETWEEN(SYSDATE, hire_date) / 12 >= 4 THEN 'Established'
    ELSE 'Junior'
  END AS tenure_band
FROM employees
ORDER BY years_tenure DESC;
SELECT
  o.region,
  o.product,
  COUNT(o.order_id)        AS total_orders,
  SUM(o.amount)            AS total_revenue,
  ROUND(AVG(o.amount), 0)  AS avg_deal_size,
  SUM(CASE WHEN o.status = 'Closed'
      THEN o.amount ELSE 0 END) AS closed_revenue
FROM sales_orders o
JOIN employees e ON o.emp_id = e.emp_id
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY o.region, o.product
ORDER BY total_revenue DESC;

SELECT
  first_name || ' ' || last_name  AS full_name,
  dept_id,
  salary,
  RANK()        OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rank_in_dept,
  DENSE_RANK()  OVER (ORDER BY salary DESC)                      AS company_rank,
  NTILE(4)      OVER (ORDER BY salary DESC)                      AS salary_quartile,
  ROUND(salary / SUM(salary) OVER (PARTITION BY dept_id) * 100, 1)
                                                                 AS pct_of_dept_payroll
FROM employees
ORDER BY dept_id, rank_in_dept;

SELECT
  e.first_name || ' ' || e.last_name  AS full_name,
  e.job_title,
  d.dept_name,
  e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary = (
  SELECT MAX(e2.salary)
  FROM employees e2
  WHERE e2.dept_id = e.dept_id
)
ORDER BY e.salary DESC;

WITH monthly AS (
  SELECT
    TO_CHAR(order_date, 'YYYY-MM')    AS month,
    SUM(amount)                        AS revenue
  FROM sales_orders
  WHERE status = 'Closed'
  GROUP BY TO_CHAR(order_date, 'YYYY-MM')
)
SELECT
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month)  AS prev_month,
  revenue - LAG(revenue) OVER (ORDER BY month)  AS mom_change,
  ROUND(
    (revenue - LAG(revenue) OVER (ORDER BY month))
    / LAG(revenue) OVER (ORDER BY month) * 100, 1
  ) AS mom_pct_change
FROM monthly
ORDER BY month;

WITH dept_payroll AS (
  SELECT
    dept_id,
    SUM(salary) AS total_salary,
    COUNT(*)    AS headcount
  FROM employees
  GROUP BY dept_id
),
budget_analysis AS (
  SELECT
    d.dept_name,
    d.location,
    d.budget,
    p.total_salary,
    p.headcount,
    d.budget - p.total_salary            AS budget_remaining,
    ROUND(p.total_salary / d.budget * 100, 1) AS payroll_pct_of_budget
  FROM departments d
  JOIN dept_payroll p ON d.dept_id = p.dept_id
)
SELECT
  dept_name,
  location,
  budget,
  total_salary,
  headcount,
  budget_remaining,
  payroll_pct_of_budget,
  CASE
    WHEN payroll_pct_of_budget > 90 THEN 'Over-allocated'
    WHEN payroll_pct_of_budget > 70 THEN 'On track'
    ELSE 'Under-utilized'
  END AS budget_status
FROM budget_analysis
ORDER BY payroll_pct_of_budget DESC;

select * from employees

WITH dept_payroll AS (
  SELECT
    dept_id,
    SUM(salary) AS total_salary,
    COUNT(*)    AS headcount
  FROM employees
  GROUP BY dept_id
),
budget_analysis AS (
  SELECT
    d.dept_name,
    d.location,
    d.budget,
    p.total_salary,
    p.headcount,
    d.budget - p.total_salary            AS budget_remaining,
    ROUND(p.total_salary / d.budget * 100, 1) AS payroll_pct_of_budget
  FROM departments d
  JOIN dept_payroll p ON d.dept_id = p.dept_id
)
SELECT
  dept_name,
  location,
  budget,
  total_salary,
  headcount,
  budget_remaining,
  payroll_pct_of_budget,
  CASE
    WHEN payroll_pct_of_budget > 90 THEN 'Over-allocated'
    WHEN payroll_pct_of_budget > 70 THEN 'On track'
    ELSE 'Under-utilized'
  END AS budget_status
FROM budget_analysis
ORDER BY payroll_pct_of_budget DESC;

SELECT
  e.first_name || ' ' || e.last_name           AS rep_name,
  d.dept_name,
  COUNT(o.order_id)                             AS total_deals,
  SUM(CASE WHEN o.status='Closed' THEN 1 END)  AS closed_deals,
  ROUND(
    SUM(CASE WHEN o.status='Closed' THEN 1 ELSE 0 END)
    / COUNT(o.order_id) * 100, 0
  )                                             AS win_rate_pct,
  SUM(CASE WHEN o.status='Closed' THEN o.amount ELSE 0 END) AS closed_revenue,
  ROUND(AVG(o.amount), 0)                       AS avg_deal_size,
  RANK() OVER (ORDER BY
    SUM(CASE WHEN o.status='Closed' THEN o.amount ELSE 0 END) DESC
  )                                             AS revenue_rank
FROM sales_orders o
JOIN employees e ON o.emp_id = e.emp_id
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY e.emp_id, e.first_name, e.last_name, d.dept_name
ORDER BY revenue_rank;

WITH org_stats AS (
  SELECT
    d.dept_name,
    COUNT(e.emp_id)                     AS headcount,
    ROUND(AVG(e.salary),0)              AS avg_salary,
    SUM(e.salary)                       AS payroll,
    d.budget,
    ROUND(SUM(e.salary)/d.budget*100,1) AS payroll_utilization
  FROM departments d
  LEFT JOIN employees e ON d.dept_id = e.dept_id
  GROUP BY d.dept_name, d.budget
),
sales_stats AS (
  SELECT
    d.dept_name,
    COUNT(o.order_id)                   AS total_orders,
    SUM(CASE WHEN o.status='Closed'
        THEN o.amount ELSE 0 END)       AS closed_rev
  FROM departments d
  LEFT JOIN employees e  ON d.dept_id  = e.dept_id
  LEFT JOIN sales_orders o ON e.emp_id = o.emp_id
  GROUP BY d.dept_name
)
SELECT
  o.dept_name,
  o.headcount,
  o.avg_salary,
  o.payroll,
  o.budget,
  o.payroll_utilization             AS payroll_pct,
  NVL(s.total_orders, 0)            AS orders,
  NVL(s.closed_rev, 0)              AS revenue,
  RANK() OVER (
    ORDER BY NVL(s.closed_rev,0) DESC
  )                                 AS revenue_rank
FROM org_stats o
LEFT JOIN sales_stats s ON o.dept_name = s.dept_name
ORDER BY o.payroll DESC;
