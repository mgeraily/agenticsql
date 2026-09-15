==============================================================================
Semantic Metadata Schema for Text-to-SQL Agent
Target Entity: budgets / v_monthly_budgets
==============================================================================

table_name: budgets
view_name: v_monthly_budgets
domain: "Sales Planning & Financial Budgeting"
description: "حاوی پیش‌بینی و بودجه فروش مقداری و ریالی کالاها/محصولات به تفکیک سال و ماه"

 مترادف‌های بیزینسی جهت Table Retrieval و Routing
synonyms:
  - "بودجه فروش"
  - "تارگت کالاها"
  - "پیش‌بینی فروش"
  - "برنامه فروش"
  - "اهداف ریالی و مقداری"
  - "هدف‌گذاری ماهانه"

 کلیدها و ایندکس‌ها
primary_keys: ["id"]
composite_business_keys: ["company_id", "sku_id", "year", "month_num"]

# ------------------------------------------------------------------------------
# پیش‌فرض‌های بیزینسی (Default Filters)
# قوانینی که کاربر مالی بیان نمی‌کند اما SQL باید رعایت کند
# ------------------------------------------------------------------------------
default_filters:
  - rule: "همیشه باید بر اساس سال مالی مورد بحث فیلتر شود؛ در صورت عدم ذکر، سال جاری پیش‌فرض است."
    sql_snippet: "year = :target_year"
  - rule: "در صورت مشخص نبودن واحد، پیش‌فرض مبلغ ریالی است."
    sql_snippet: "metric_type = 'amount'"

# ------------------------------------------------------------------------------
# مسیرهای ارتباطی (Join Paths)
# ------------------------------------------------------------------------------
join_paths:
  - target_table: skus
    join_type: "INNER JOIN"
    on: "v_monthly_budgets.sku_id = skus.id"
    description: "ارتباط به اطلاعات تکمیلی کالاها، برندها و مدل‌ها"
  - target_table: companies
    join_type: "INNER JOIN"
    on: "v_monthly_budgets.company_id = companies.id"
    description: "ارتباط به اطلاعات شرکت تابعه"
  - target_table: cost_center
    join_type: "LEFT JOIN"
    on: "skus.cost_center_id = cost_center.id"
    description: "ارتباط به مرکز هزینه یا خط تولید مرتبط"

# ------------------------------------------------------------------------------
# متادیتای ستون‌ها و قوانین تجمیع (Columns & Aggregation Rules)
# ------------------------------------------------------------------------------
columns:
  - name: id
    type: "INT"
    description: "شناسه یکتا رکورد بودجه"
    is_filterable: false

  - name: company_id
    type: "INT"
    description: "شناسه شرکت مالک بودجه"
    join_target: "companies.id"
    entity_resolution_field: "companies.title"

  - name: sku_id
    type: "INT"
    description: "شناسه کالا یا محصول"
    join_target: "skus.id"
    entity_resolution_field: "skus.title"

  - name: year
    type: "INT"
    description: "سال مالی خورشیدی (مانند 1402, 1403, 1404)"
    range: [1390, 1450]

  - name: month_num
    type: "INT"
    description: "شماره ماه از 1 (فروردین) تا 12 (اسفند)"
    range: [1, 12]

  - name: metric_type
    type: "VARCHAR(20)"
    description: "نوع بودجه: مبلغ ریالی یا تعداد/حجم"
    allowed_values:
      amount: "مبلغ ریالی بودجه (ریال)"
      quantity: "تعداد، وزن یا حجم کالا"

  - name: amount
    type: "BIGINT / DECIMAL"
    description: "مقدار بودجه ثبت‌شده (ریالی یا تعدادی بر اساس metric_type)"
    unit: "IRR (در صورت amount) / Count/Kg (در صورت quantity)"
    aggregation_rules:
      can_sum: true
      can_avg: true
      default_aggregation: "SUM"

  - name: driver
    type: "VARCHAR(50)"
    description: "منطق و متدولوژی برآورد بودجه"
    allowed_values:
      manual: "تکمیل دستی توسط مدیران محصول"
      formula: "محاسبه بر مبنای نرخ رشد سالانه"
      channel_share: "محاسبه بر اساس سهم کانال‌های توزیع"

# ------------------------------------------------------------------------------
# ستون‌های دارای مقادیر شمارشی و نگاشت اصطلاحات کاربر (Enums & Aliases)
# ------------------------------------------------------------------------------
categorical_columns:
  - column: metric_type
    exact_db_values:
      - db_value: "amount"
        persian_label: "مبلغ ریالی"
        user_aliases: ["ریالی", "مبلغ", "ارزش", "فروش ریالی", "تومان", "پول"]
      - db_value: "quantity"
        persian_label: "تعداد / مقداری"
        user_aliases: ["تعدادی", "حجمی", "تعداد", "دستگاه", "عدد", "تیراژ"]

  - column: driver
    exact_db_values:
      - db_value: "manual"
        persian_label: "دستی"
        user_aliases: ["دستی", "وارد شده", "ثبت مستقیم"]
      - db_value: "formula"
        persian_label: "فرمول / محاسباتی"
        user_aliases: ["فرمول", "محاسباتی", "سیستمی", "رشد"]
      - db_value: "channel_share"
        persian_label: "سهم کانال"
        user_aliases: ["کانال توزیع", "سهم کانال", "نمایندگی"]

# ------------------------------------------------------------------------------
# داده‌های نمونه ابعادی متصل (Dimension Samples & Entity Linking)
# ------------------------------------------------------------------------------
dimension_samples:
  - dimension_name: "brand"
    table_source: "skus"
    target_column: "skus.brand_title"
    sample_values_in_db:
      - "Scania"
      - "Foton"
      - "Mammut"
      - "Shacman"
    user_aliases_mapping:
      "اسکانیا": "Scania"
      "فوتون": "Foton"
      "ماموت": "Mammut"
      "شکمان": "Shacman"
      "شاکمن": "Shacman"

  - dimension_name: "vehicle_type"
    table_source: "skus"
    target_column: "skus.category_code"
    sample_values_in_db:
      - "Trailer_Curtain"
      - "Trailer_Reefer"
      - "Truck_Tractor"
      - "Tipper"
    user_aliases_mapping:
      "چادری": "Trailer_Curtain"
      "ترانزیت": "Trailer_Curtain"
      "یخچالی": "Trailer_Reefer"
      "سردخانه‌ای": "Trailer_Reefer"
      "کشنده": "Truck_Tractor"
      "تریلی": "Truck_Tractor"
      "کمپرسی": "Tipper"

# ------------------------------------------------------------------------------
# رکوردهای نمونه واقعی از دیتابیس (Representative Sample Rows)
# ------------------------------------------------------------------------------
representative_rows:
  - id: 1001
    company_id: 1
    sku_id: 501
    year: 1403
    month_num: 1
    metric_type: "amount"
    amount: 25000000000 # 25 میلیارد ریال
    driver: "formula"

  - id: 1002
    company_id: 1
    sku_id: 501
    year: 1403
    month_num: 1
    metric_type: "quantity"
    amount: 15 # 15 دستگاه
    driver: "formula"

  - id: 1003
    company_id: 1
    sku_id: 502
    year: 1403
    month_num: 2
    metric_type: "amount"
    amount: 18000000000
    driver: "manual"

# ------------------------------------------------------------------------------
# راهنما و خط قرمزهای تولید کوئری برای LLM (Guardrails & Anti-Patterns)
# ------------------------------------------------------------------------------
guardrails:
  - "روی جدول افقی budgets کوئری مستقیم نزنید؛ همواره از View با نام v_monthly_budgets استفاده کنید چون ساختار ماهانه را Unpivot کرده است."
  - "هرگز amount با نوع 'amount' و 'quantity' را بدون تفکیک با هم SUM نکنید؛ شرط metric_type اجباری است."
  - "اگر کاربر سوالی درباره فصل‌ها پرسید، نگاشت ماه‌ها بدین صورت است: بهار (1,2,3)، تابستان (4,5,6)، پاییز (7,8,9)، زمستان (10,11,12)."
  - "در صورت مقایسه دو سال مالی، از JOIN روی sku_id و month_num استفاده کنید و به ساختار Pivot دست نزنید."

# ------------------------------------------------------------------------------
# پرسش‌های پرتکرار جهت Few-Shot و ارزیابی (Sample / Benchmark Questions)
# ------------------------------------------------------------------------------
sample_questions:
  - question: "جمع بودجه ریالی فروش کشنده‌ها در سه ماهه اول 1403 چقدر است؟"
    expected_tables: ["v_monthly_budgets", "skus"]
    target_metric: "amount"
  - question: "تارگت مقداری (تعدادی) کالای فوتون در سال 1402 چقدر ثبت شده بود؟"
    expected_tables: ["v_monthly_budgets", "skus"]
    target_metric: "quantity"
