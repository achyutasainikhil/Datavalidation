# Fabric notebook (PySpark). Profiles the two Party Core bronze sources.
# Run each cell in order. Paste the outputs of cells 2, 3 and 4 back to Claude.
# Adjust the table names if your lakehouse paths differ.
from pyspark.sql import functions as F
 
TABLES = {
    "rtmi": "lh_bronze.mpi_bronze.rtmi_individual",
    "iris": "lh_bronze.mpi_bronze.iris_customer",
}
# columns that hold code values: list every distinct value (cell 3)
CODE_COLS = {
    "rtmi": ["DELTA_TYPE","DECEASED_INDICATOR","GENDER","MARITAL_STATUS","ADDRESS_TYPE","STATE",
             "ADDRESS_CLEANSED_FLAG","ADDRESS_DELIVERABLE_FLAG","NCOA_ACTION","NCOA_DELIVERY",
             "NCOA_MOVE_TYPE","WEALTH_CODE","OPT_OUT_REASON","DOB_SOURCE_INDICATOR","COUNTY"],
    "iris": ["Event_type","STATE"],
}
 
# ---- Cell 2: per-column profile (nulls, blanks, distinct, min, max) ----
def profile(key):
    df = spark.table(TABLES[key]); n = df.count(); rows = []
    for c, t in df.dtypes:
        s = F.col(f"`{c}`")
        agg = df.agg(
            F.sum(F.when(s.isNull(), 1).otherwise(0)).alias("nulls"),
            F.sum(F.when(F.trim(s.cast("string")) == "", 1).otherwise(0)).alias("blanks"),
            F.countDistinct(s).alias("distinct"),
            F.min(s.cast("string")).alias("min"),
            F.max(s.cast("string")).alias("max"),
            F.max(F.length(s.cast("string"))).alias("max_len"),
        ).collect()[0]
        rows.append((key, c, t, n, agg["nulls"], round(100.0*agg["nulls"]/n, 2) if n else None,
                     agg["blanks"], agg["distinct"], agg["min"], agg["max"], agg["max_len"]))
    cols = ["source","column","type","rows","nulls","null_pct","blanks","distinct","min","max","max_len"]
    return spark.createDataFrame(rows, cols)
for k in TABLES:
    display(profile(k).orderBy("column"))
 
# ---- Cell 3: distinct values of the code columns (top 50 each) ----
for k, cs in CODE_COLS.items():
    df = spark.table(TABLES[k])
    for c in cs:
        if c in df.columns:
            print(f"== {k}.{c}")
            display(df.groupBy(F.col(f"`{c}`")).count().orderBy(F.desc("count")).limit(50))
 
# ---- Cell 4: keys, formats, ranges ----
r = spark.table(TABLES["rtmi"]); i = spark.table(TABLES["iris"])
def show(label, df): print("==", label); display(df)
# keys
show("RTMI duplicate member numbers per file", r.groupBy("File_Date","RTMI_MEMBER_NUMBER").count().filter("count>1").groupBy().agg(F.count("*").alias("dup_keys"), F.sum("count").alias("dup_rows")))
show("RTMI null/blank member number", r.filter(F.col("RTMI_MEMBER_NUMBER").isNull() | (F.trim(F.col("RTMI_MEMBER_NUMBER").cast("string"))=="")).groupBy().count())
show("Iris duplicate IRIS_ID per IngestedTime", i.groupBy("IRIS_ID","IngestedTime").count().filter("count>1").groupBy().agg(F.count("*").alias("dup_keys"), F.sum("count").alias("dup_rows")))
show("Iris null/blank IRIS_ID", i.filter(F.col("IRIS_ID").isNull() | (F.trim(F.col("IRIS_ID").cast("string"))=="")).groupBy().count())
# SSN: raw-SSN check FIRST (token column must never hold a raw SSN)
show("RTMI SSN_TOKEN shaped like a raw SSN", r.filter(F.col("SSN_TOKEN").rlike(r"^[0-9]{3}-?[0-9]{2}-?[0-9]{4}$")).groupBy().count())
show("Iris SSN_TOKEN shaped like a raw SSN", i.filter(F.col("SSN_TOKEN").rlike(r"^[0-9]{3}-?[0-9]{2}-?[0-9]{4}$")).groupBy().count())
show("Iris SSN_LAST_FOUR not 4 digits", i.filter(F.col("SSN_LAST_FOUR").isNotNull() & ~F.col("SSN_LAST_FOUR").cast("string").rlike(r"^[0-9]{4}$")).groupBy().count())
show("Iris SSN_LAST_FOUR present with no SSN_TOKEN", i.filter(F.col("SSN_LAST_FOUR").isNotNull() & F.col("SSN_TOKEN").isNull()).groupBy().count())
# ZIP / lat / long / phone
show("RTMI ZIP not 5 or 5+4", r.filter(F.col("ZIP_CODE").isNotNull() & ~F.col("ZIP_CODE").rlike(r"^[0-9]{5}(-?[0-9]{4})?$")).groupBy().count())
show("Iris ZIP not 5 or 5+4", i.filter(F.col("ZIP_CODE").isNotNull() & ~F.col("ZIP_CODE").rlike(r"^[0-9]{5}(-?[0-9]{4})?$")).groupBy().count())
show("RTMI latitude non-numeric or out of range", r.filter(F.col("LATITUDE").isNotNull() & (F.col("LATITUDE").cast("double").isNull() | (F.abs(F.col("LATITUDE").cast("double"))>90))).groupBy().count())
show("RTMI longitude non-numeric or out of range", r.filter(F.col("LONGITUDE").isNotNull() & (F.col("LONGITUDE").cast("double").isNull() | (F.abs(F.col("LONGITUDE").cast("double"))>180))).groupBy().count())
for c in ["HOME_TELEPHONE_NUMBER","BUSINESS_TELEPHONE_NUMBER"]:
    d = F.regexp_replace(F.col(c), r"[^0-9]", "")
    show(f"RTMI {c} digit-length distribution", r.filter(F.col(c).isNotNull()).groupBy(F.length(d).alias("digits")).count().orderBy("digits"))
# dates
show("RTMI DOB range / future", r.agg(F.min("DATE_OF_BIRTH"), F.max("DATE_OF_BIRTH"), F.sum(F.when(F.col("DATE_OF_BIRTH")>F.current_date(),1).otherwise(0)).alias("future"), F.sum(F.when(F.col("DATE_OF_BIRTH")<F.lit("1900-01-01").cast("date"),1).otherwise(0)).alias("before_1900")))
show("RTMI DOD < DOB or future", r.agg(F.sum(F.when(F.col("DATE_OF_DEATH")<F.col("DATE_OF_BIRTH"),1).otherwise(0)).alias("dod_before_dob"), F.sum(F.when(F.col("DATE_OF_DEATH")>F.current_date(),1).otherwise(0)).alias("dod_future")))
show("Iris BIRTH_DATE range / future", i.agg(F.min("BIRTH_DATE"), F.max("BIRTH_DATE"), F.sum(F.when(F.col("BIRTH_DATE")>F.current_date(),1).otherwise(0)).alias("future"), F.sum(F.when(F.col("BIRTH_DATE")<F.lit("1900-01-01").cast("date"),1).otherwise(0)).alias("before_1900")))
show("Iris EndDate vs StartDate", i.agg(F.count("StartDate").alias("start_nonnull"), F.count("EndDate").alias("end_nonnull"), F.sum(F.when(F.col("EndDate")<F.col("StartDate"),1).otherwise(0)).alias("end_before_start")))
show("RTMI ORIGIN_TIMESTAMP null / future", r.agg(F.sum(F.when(F.col("ORIGIN_TIMESTAMP").isNull(),1).otherwise(0)).alias("null"), F.sum(F.when(F.col("ORIGIN_TIMESTAMP")>F.current_timestamp(),1).otherwise(0)).alias("future")))
 
