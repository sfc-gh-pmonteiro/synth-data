# SYNTH_USE_CASE_SCHEMA_MAP — Vertical Use Cases → Star Schema Definitions
<!-- Maps HIGH-impact use cases from 12 verticals to Snowflake star-schema definitions with distribution parameters. -->
<!-- Reference for synthetic data generation. Each entry: Fact, Dims, Grain, Volume, Distributions, Seasonality. -->

## Field Reference

Each use case entry contains these fields:
- **Fact**: Table name and column definitions with FK references
- **Dims**: Dimension tables and their key columns
- **Grain**: What one row represents
- **Volume**: Rows/year at medium scale (small = /10, large = *10)
- **Distributions**: Distribution function per column (ZIPF, NORMAL, UNIFORM, Pattern#11)
- **Seasonality**: Named seasonal patterns with multipliers
- **Dim Sizes**: Row counts per dimension table at medium scale (small = /5, large = *5)
- **Intraday** *(optional)*: Pattern #13 hour profile for timestamp generation
- **Correlated FK** *(optional)*: FK derivation rule (child→parent via JOIN)
- **Note** *(optional)*: Region-specific or implementation guidance

## Healthcare > Patient 360
- **Fact**: `fact_encounter` (encounter_id INT, patient_id INT FK→dim_patient, provider_id INT FK→dim_provider, date_id INT FK→dim_date, charge_amount NUMBER(12,2), los_days NUMBER(4,0), diagnosis_count INT, status VARCHAR)
- **Dims**: `dim_date`, `dim_patient`(patient_id, name, dob, gender, zip), `dim_provider`(provider_id, npi, specialty, facility)
- **Grain**: One row per patient encounter
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: charge_amount=NORMAL(2200,3500), los_days=ZIPF(2.0), patient_id=ZIPF(1.1), provider_id=ZIPF(1.1)
- **Dim Sizes**: dim_patient=500, dim_provider=100
- **Seasonality**: winter_respiratory(Dec-Feb, +1.3x), summer_dip(Jun-Aug, 0.85x)

## Healthcare > Claims Analytics & Fraud Detection
- **Fact**: `fact_claim` (claim_id INT, member_id INT FK→dim_member, provider_id INT FK→dim_provider, procedure_id INT FK→dim_procedure, date_id INT FK→dim_date, billed_amount NUMBER(12,2), paid_amount NUMBER(12,2), fraud_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_member`(member_id, plan_type, age_band, state), `dim_provider`(provider_id, npi, specialty, network_status), `dim_procedure`(procedure_id, cpt_code, category)
- **Grain**: One row per claim line
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: billed_amount=NORMAL(850,2500), paid_amount=NORMAL(620,1800), provider_id=ZIPF(1.1), fraud_flag=UNIFORM(0.03)
- **Dim Sizes**: dim_member=1000, dim_provider=200, dim_procedure=300
- **Seasonality**: year_end_rush(Dec, +1.2x), Q1_deductible_reset(Jan, +1.15x)

## Healthcare > Readmission Risk
- **Fact**: `fact_admission` (admission_id INT, patient_id INT FK→dim_patient, date_id INT FK→dim_date, los_days NUMBER(4,0), total_cost NUMBER(12,2), readmit_30d BOOLEAN, risk_score NUMBER(5,3))
- **Dims**: `dim_date`, `dim_patient`(patient_id, age, chronic_count, sdoh_index), `dim_diagnosis`(diagnosis_id, icd_code, category, severity)
- **Grain**: One row per inpatient admission
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: los_days=ZIPF(1.8), total_cost=NORMAL(15000,12000), readmit_30d=UNIFORM(0.14), patient_id=ZIPF(1.1)
- **Dim Sizes**: dim_patient=400, dim_diagnosis=200
- **Seasonality**: flu_season(Nov-Feb, +1.25x), holiday_dip(Dec 20-Jan 2, −0.3x elective)

## Healthcare > Population Health
- **Fact**: `fact_member_risk` (member_risk_id INT, member_id INT FK→dim_member, date_id INT FK→dim_date, hcc_score NUMBER(6,3), total_cost NUMBER(12,2), er_visits INT, rx_adherence_pct NUMBER(5,2))
- **Dims**: `dim_date`, `dim_member`(member_id, age_band, gender, chronic_conditions, zip), `dim_risk_segment`(segment_id, name, intervention_type)
- **Grain**: One row per member per month
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: hcc_score=ZIPF(1.6), total_cost=ZIPF(1.9), er_visits=ZIPF(2.5), member_id=UNIFORM
- **Dim Sizes**: dim_member=600, dim_risk_segment=10
- **Seasonality**: respiratory_season(Nov-Mar, +1.2x cost), summer_low(Jun-Aug, 0.9x)

## Retail & CPG > Customer 360
- **Fact**: `fact_transaction` (txn_id INT, customer_id INT FK→dim_customer, store_id INT FK→dim_store, date_id INT FK→dim_date, basket_amount NUMBER(10,2), item_count INT, channel VARCHAR, payment_method VARCHAR, transaction_timestamp TIMESTAMP)
- **Dims**: `dim_date`, `dim_customer`(customer_id, loyalty_tier, signup_date, zip), `dim_store`(store_id, name, region, format), `dim_product`(product_id, sku, name, category, subcategory, unit_price NUMBER(8,2))
- **Grain**: One row per transaction
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: basket_amount=NORMAL(45,30), customer_id=ZIPF(1.1), store_id=ZIPF(1.1), item_count=NORMAL(8,5), payment_method=Pattern#11(credit:0.45,debit:0.30,cash:0.15,digital:0.10)
- **Dim Sizes**: dim_customer=1000, dim_store=50, dim_product=500
- **Seasonality**: holiday_spike(Nov-Dec, +1.5x), post_holiday_dip(Jan, 0.75x), back_to_school(Aug, +1.2x)
- **Intraday**: Pattern #13 retail bimodal (peaks 12-14h, 18-20h)
- **Note**: dim_product categories should be region-appropriate. For LATAM: Abarrotes, Lácteos, Carnes, Bebidas, Frutas y Verduras, Limpieza, Perfumería, Tecnología. For US/EU: Grocery, Dairy, Meat, Beverages, Produce, Household, Personal Care, Electronics.

## Retail & CPG > Demand Forecasting
- **Fact**: `fact_daily_sales` (sale_id INT, product_id INT FK→dim_product, store_id INT FK→dim_store, date_id INT FK→dim_date, units_sold NUMBER(8,0), revenue NUMBER(10,2), promo_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_product`(product_id, sku, category, brand, shelf_life_days), `dim_store`(store_id, region, format, sqft)
- **Grain**: One row per product per store per day
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: units_sold=NORMAL(12,8), revenue=NORMAL(35,25), product_id=ZIPF(1.1), promo_flag=UNIFORM(0.15)
- **Dim Sizes**: dim_product=500, dim_store=50
- **Seasonality**: holiday_peak(Nov-Dec, +1.4x), summer_beverages(Jun-Aug, +1.3x category-specific)

## Retail & CPG > Price Optimization
- **Fact**: `fact_price_change` (price_change_id INT, product_id INT FK→dim_product, date_id INT FK→dim_date, old_price NUMBER(8,2), new_price NUMBER(8,2), units_before NUMBER(8,0), units_after NUMBER(8,0), elasticity NUMBER(5,3))
- **Dims**: `dim_date`, `dim_product`(product_id, category, brand, cost), `dim_competitor`(competitor_id, name, channel)
- **Grain**: One row per price change event per product
- **Volume**: ~50K rows/year at medium scale
- **Distributions**: elasticity=NORMAL(-1.5,0.8), old_price=NORMAL(12,8), product_id=ZIPF(1.1)
- **Dim Sizes**: dim_product=200, dim_competitor=20
- **Seasonality**: markdown_season(Jan+Jul, +3x events), pre_holiday_lock(Nov, −0.5x changes)

## Retail & CPG > Personalized Recommendations
- **Fact**: `fact_interaction` (interaction_id INT, customer_id INT FK→dim_customer, product_id INT FK→dim_product, date_id INT FK→dim_date, event_type VARCHAR, score NUMBER(5,3), converted BOOLEAN)
- **Dims**: `dim_date`, `dim_customer`(customer_id, segment, preference_vector), `dim_product`(product_id, category, attributes)
- **Grain**: One row per customer-product interaction event
- **Volume**: ~800K rows/year at medium scale
- **Distributions**: customer_id=ZIPF(1.1), product_id=ZIPF(1.1), score=NORMAL(0.35,0.2), converted=UNIFORM(0.04)
- **Dim Sizes**: dim_customer=1000, dim_product=500
- **Seasonality**: browsing_surge(Nov-Dec, +2.0x), summer_lull(Jul, 0.8x)

## Financial Services > Customer 360
- **Fact**: `fact_transaction` (txn_id INT, account_id INT FK→dim_account, customer_id INT FK→dim_customer, date_id INT FK→dim_date, amount NUMBER(14,2), txn_type VARCHAR, channel VARCHAR, transaction_timestamp TIMESTAMP)
- **Dims**: `dim_date`, `dim_customer`(customer_id, segment, relationship_start, household_id), `dim_account`(account_id, product_type, balance_tier, branch_id, customer_id INT FK→dim_customer)
- **Grain**: One row per financial transaction
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: amount=ZIPF(1.9), customer_id=ZIPF(1.1), account_id=ZIPF(1.1)
- **Dim Sizes**: dim_customer=500, dim_account=800
- **Seasonality**: tax_season(Mar-Apr, +1.3x), year_end(Dec, +1.2x), summer_flat(Jul-Aug, 0.9x)
- **Intraday**: Pattern #13 business hours (peak 9-16h)
- **Correlated FK**: account_id → dim_account → customer_id (derive customer from account, never generate independently)

## Financial Services > Fraud Detection
- **Fact**: `fact_txn_scored` (score_id INT, txn_id INT, customer_id INT FK→dim_customer, date_id INT FK→dim_date, amount NUMBER(14,2), fraud_score NUMBER(5,3), is_fraud BOOLEAN, channel VARCHAR)
- **Dims**: `dim_date`, `dim_customer`(customer_id, account_age_days, avg_txn_amount), `dim_device`(device_id, fingerprint, geo_region, is_known)
- **Grain**: One row per scored transaction
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: amount=ZIPF(2.0), fraud_score=NORMAL(0.12,0.15), is_fraud=UNIFORM(0.008), customer_id=ZIPF(1.1)
- **Dim Sizes**: dim_customer=800, dim_device=300
- **Seasonality**: holiday_fraud_spike(Nov-Dec, +1.8x fraud rate), tax_refund_fraud(Feb-Apr, +1.4x)

## Financial Services > AML/KYC
- **Fact**: `fact_alert` (alert_id INT, customer_id INT FK→dim_customer, date_id INT FK→dim_date, alert_type VARCHAR, amount_involved NUMBER(14,2), risk_score NUMBER(5,3), disposition VARCHAR)
- **Dims**: `dim_date`, `dim_customer`(customer_id, kyc_status, country_risk, pep_flag), `dim_rule`(rule_id, name, threshold, category)
- **Grain**: One row per AML alert generated
- **Volume**: ~30K rows/year at medium scale
- **Distributions**: amount_involved=ZIPF(1.8), risk_score=NORMAL(0.4,0.25), customer_id=ZIPF(1.1)
- **Dim Sizes**: dim_customer=500, dim_rule=50
- **Seasonality**: quarter_end_filing(Mar/Jun/Sep/Dec, +1.5x), year_end_review(Dec, +1.3x)

## Financial Services > Credit Risk
- **Fact**: `fact_application` (app_id INT, customer_id INT FK→dim_customer, date_id INT FK→dim_date, requested_amount NUMBER(12,2), credit_score INT, pd_score NUMBER(5,4), decision VARCHAR)
- **Dims**: `dim_date`, `dim_customer`(customer_id, income_band, employment_type, existing_debt), `dim_product`(product_id, loan_type, term_months)
- **Grain**: One row per credit application
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: requested_amount=NORMAL(25000,18000), credit_score=NORMAL(710,65), pd_score=ZIPF(2.0), customer_id=ZIPF(1.1)
- **Dim Sizes**: dim_customer=500, dim_product=20
- **Seasonality**: spring_auto(Mar-May, +1.2x auto), back_to_school(Aug, +1.3x personal), holiday_credit(Nov-Dec, +1.4x)

## Insurance > Claims Analytics & Fraud
- **Fact**: `fact_claim` (claim_id INT, policy_id INT FK→dim_policy, claimant_id INT FK→dim_claimant, date_id INT FK→dim_date, incurred_amount NUMBER(12,2), paid_amount NUMBER(12,2), fraud_score NUMBER(5,3), status VARCHAR)
- **Dims**: `dim_date`, `dim_policy`(policy_id, lob, coverage_type, effective_date), `dim_claimant`(claimant_id, state, claim_history_count)
- **Grain**: One row per claim
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: incurred_amount=ZIPF(1.7), paid_amount=ZIPF(1.8), fraud_score=NORMAL(0.15,0.18), policy_id=ZIPF(1.1)
- **Dim Sizes**: dim_policy=1000, dim_claimant=500
- **Seasonality**: winter_auto(Dec-Feb, +1.4x auto), spring_property(Mar-May, +1.3x hail/wind), hurricane(Aug-Oct, +2.0x cat)

## Insurance > Underwriting Automation
- **Fact**: `fact_submission` (submission_id INT, applicant_id INT FK→dim_applicant, date_id INT FK→dim_date, premium NUMBER(10,2), risk_score NUMBER(5,3), decision VARCHAR, stp_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_applicant`(applicant_id, industry, revenue_band, state), `dim_product`(product_id, lob, class_code)
- **Grain**: One row per underwriting submission
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: premium=ZIPF(1.9), risk_score=NORMAL(0.5,0.2), stp_flag=UNIFORM(0.6), applicant_id=ZIPF(1.1)
- **Dim Sizes**: dim_applicant=500, dim_product=30
- **Seasonality**: renewal_cycle(Jan+Jul, +1.5x), year_end_binding(Dec, +1.3x)

## Insurance > Pricing & Rate Adequacy
- **Fact**: `fact_policy_period` (period_id INT, policy_id INT FK→dim_policy, date_id INT FK→dim_date, earned_premium NUMBER(10,2), incurred_loss NUMBER(12,2), exposure_units NUMBER(8,2), loss_ratio NUMBER(5,3))
- **Dims**: `dim_date`, `dim_policy`(policy_id, lob, territory, class_code), `dim_rate_group`(rate_group_id, factor_set, effective_date)
- **Grain**: One row per policy per earned month
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: earned_premium=NORMAL(800,500), incurred_loss=ZIPF(1.6), loss_ratio=NORMAL(0.65,0.2)
- **Dim Sizes**: dim_policy=800, dim_rate_group=50
- **Seasonality**: cat_season(Jun-Oct, +1.8x losses), mild_winter(Jan-Mar, 0.9x auto)

## Insurance > Catastrophe Modeling
- **Fact**: `fact_exposure` (exposure_id INT, policy_id INT FK→dim_policy, location_id INT FK→dim_location, date_id INT FK→dim_date, tiv NUMBER(14,2), pml NUMBER(14,2), peril VARCHAR)
- **Dims**: `dim_date`, `dim_policy`(policy_id, lob, deductible, limit), `dim_location`(location_id, lat, lon, construction_type, flood_zone, wind_zone)
- **Grain**: One row per insured location per peril
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: tiv=ZIPF(1.7), pml=ZIPF(1.9), location_id=ZIPF(1.1)
- **Dim Sizes**: dim_policy=600, dim_location=500
- **Seasonality**: hurricane_season(Jun-Nov, +2.5x attention), renewal_accumulation(Jan+Jul, +1.3x new exposures)

## Technology/SaaS > Product-Led Growth
- **Fact**: `fact_product_event` (event_id INT, user_id INT FK→dim_user, feature_id INT FK→dim_feature, date_id INT FK→dim_date, event_type VARCHAR, session_duration_sec NUMBER(8,0), is_activation BOOLEAN)
- **Dims**: `dim_date`, `dim_user`(user_id, signup_date, plan, source_channel), `dim_feature`(feature_id, name, category, tier_required)
- **Grain**: One row per product interaction event
- **Volume**: ~800K rows/year at medium scale
- **Distributions**: user_id=ZIPF(1.1), feature_id=ZIPF(1.1), session_duration_sec=NORMAL(320,200)
- **Dim Sizes**: dim_user=1000, dim_feature=50
- **Seasonality**: q1_new_year_signups(Jan, +1.4x), summer_dip(Jul-Aug, 0.85x), q4_budget_push(Oct-Nov, +1.2x)

## Technology/SaaS > Customer Health & Churn
- **Fact**: `fact_health_score` (score_id INT, account_id INT FK→dim_account, date_id INT FK→dim_date, health_score NUMBER(5,2), login_count INT, support_tickets INT, usage_pct NUMBER(5,2))
- **Dims**: `dim_date`, `dim_account`(account_id, tier, arr_band, csm_id, contract_end_date), `dim_segment`(segment_id, name, risk_level)
- **Grain**: One row per account per week
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: health_score=NORMAL(72,18), login_count=ZIPF(1.8), support_tickets=ZIPF(2.5), usage_pct=NORMAL(0.6,0.25)
- **Dim Sizes**: dim_account=500, dim_segment=10
- **Seasonality**: post_holiday_churn(Jan-Feb, +1.5x churn risk), renewal_season(varies by contract)

## Technology/SaaS > Usage-Based Billing
- **Fact**: `fact_meter_event` (meter_id INT, tenant_id INT FK→dim_tenant, date_id INT FK→dim_date, api_calls NUMBER(10,0), compute_units NUMBER(10,2), storage_gb NUMBER(8,2), overage_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_tenant`(tenant_id, plan_tier, commitment_amount, region), `dim_service`(service_id, name, unit_type, rate_per_unit)
- **Grain**: One row per tenant per service per day
- **Volume**: ~350K rows/year at medium scale
- **Distributions**: api_calls=ZIPF(1.7), compute_units=ZIPF(1.9), tenant_id=ZIPF(1.1), storage_gb=NORMAL(50,80)
- **Dim Sizes**: dim_tenant=200, dim_service=20
- **Seasonality**: month_end_batch(last 3 days, +2.0x), q4_spike(Nov-Dec, +1.3x)

## Technology/SaaS > Revenue Intelligence (ARR/MRR)
- **Fact**: `fact_arr_movement` (movement_id INT, account_id INT FK→dim_account, date_id INT FK→dim_date, movement_type VARCHAR, arr_amount NUMBER(12,2), prev_arr NUMBER(12,2), rep_id INT FK→dim_rep)
- **Dims**: `dim_date`, `dim_account`(account_id, segment, industry, region), `dim_rep`(rep_id, name, territory, quota)
- **Grain**: One row per ARR change event (new, expansion, contraction, churn)
- **Volume**: ~50K rows/year at medium scale
- **Distributions**: arr_amount=ZIPF(1.9), account_id=ZIPF(1.1), rep_id=ZIPF(1.1)
- **Dim Sizes**: dim_account=300, dim_rep=50
- **Seasonality**: q4_close(Dec, +2.0x new), q1_contraction(Jan-Feb, +1.5x churn), fiscal_year_end(varies, +1.8x)

## Technology/SaaS > Trial-to-Paid
- **Fact**: `fact_trial` (trial_id INT, user_id INT FK→dim_user, date_id INT FK→dim_date, trial_day INT, features_used INT, converted BOOLEAN, plan_selected VARCHAR)
- **Dims**: `dim_date`, `dim_user`(user_id, source, company_size, role), `dim_plan`(plan_id, name, price, feature_set)
- **Grain**: One row per trial user per day during trial
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: features_used=ZIPF(1.8), user_id=UNIFORM, converted=UNIFORM(0.12)
- **Dim Sizes**: dim_user=1000, dim_plan=5
- **Seasonality**: new_year_trials(Jan, +1.6x signups), conference_bump(event-driven, +1.3x)

## Manufacturing > Predictive Maintenance
- **Fact**: `fact_sensor_reading` (reading_id INT, equipment_id INT FK→dim_equipment, date_id INT FK→dim_date, vibration NUMBER(8,3), temperature NUMBER(6,2), pressure NUMBER(8,2), anomaly_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_equipment`(equipment_id, type, install_date, location, criticality), `dim_maintenance`(maintenance_id, type, cost, downtime_hrs)
- **Grain**: One row per equipment per hour
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: vibration=NORMAL(2.5,0.8), temperature=NORMAL(72,12), equipment_id=UNIFORM, anomaly_flag=UNIFORM(0.02)
- **Dim Sizes**: dim_equipment=100, dim_maintenance=50
- **Seasonality**: summer_heat(Jun-Aug, +1.2x failures), shutdown_maintenance(Dec, −0.5x readings)

## Manufacturing > Quality Analytics
- **Fact**: `fact_inspection` (inspection_id INT, lot_id INT FK→dim_lot, equipment_id INT FK→dim_equipment, date_id INT FK→dim_date, defect_count INT, first_pass BOOLEAN, rework_cost NUMBER(8,2))
- **Dims**: `dim_date`, `dim_lot`(lot_id, material_id, batch_size, supplier_id), `dim_equipment`(equipment_id, line, station), `dim_defect_type`(defect_type_id, name, severity)
- **Grain**: One row per lot inspection
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: defect_count=ZIPF(2.2), rework_cost=NORMAL(350,500), lot_id=UNIFORM, first_pass=UNIFORM(0.92)
- **Dim Sizes**: dim_lot=500, dim_equipment=50, dim_defect_type=20
- **Seasonality**: new_material_lots(Q1, +1.2x defects), post_shutdown(Jan, +1.3x startup defects)

## Manufacturing > OEE Optimization
- **Fact**: `fact_production_run` (run_id INT, machine_id INT FK→dim_machine, date_id INT FK→dim_date, planned_minutes INT, actual_minutes INT, good_count INT, reject_count INT, downtime_minutes INT)
- **Dims**: `dim_date`, `dim_machine`(machine_id, line, type, max_speed, install_year), `dim_stop_reason`(stop_id, category, planned_flag)
- **Grain**: One row per machine per shift
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: downtime_minutes=ZIPF(2.0), good_count=NORMAL(450,80), reject_count=ZIPF(2.5), machine_id=UNIFORM
- **Dim Sizes**: dim_machine=60, dim_stop_reason=25
- **Seasonality**: changeover_spike(product_transition, +0.3x downtime), maintenance_windows(quarterly, −0.2x capacity)

## Manufacturing > Supply Chain
- **Fact**: `fact_po_delivery` (delivery_id INT, supplier_id INT FK→dim_supplier, material_id INT FK→dim_material, date_id INT FK→dim_date, order_qty NUMBER(10,0), received_qty NUMBER(10,0), lead_days INT, unit_cost NUMBER(8,4))
- **Dims**: `dim_date`, `dim_supplier`(supplier_id, name, country, rating), `dim_material`(material_id, name, category, reorder_point)
- **Grain**: One row per PO delivery receipt
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: lead_days=NORMAL(14,7), unit_cost=ZIPF(1.6), supplier_id=ZIPF(1.1)
- **Dim Sizes**: dim_supplier=200, dim_material=300
- **Seasonality**: year_end_stockpile(Nov-Dec, +1.4x), chinese_new_year(Feb, +2.0x lead_days for APAC suppliers)

## Telecom > Customer 360
- **Fact**: `fact_usage` (usage_id INT, subscriber_id INT FK→dim_subscriber, date_id INT FK→dim_date, data_mb NUMBER(10,2), voice_min NUMBER(8,0), sms_count INT, arpu NUMBER(8,2))
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, plan_type, tenure_months, handset, region), `dim_plan`(plan_id, name, price, data_cap)
- **Grain**: One row per subscriber per day
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: data_mb=ZIPF(1.6), voice_min=NORMAL(25,20), subscriber_id=UNIFORM, arpu=NORMAL(45,20)
- **Dim Sizes**: dim_subscriber=200, dim_plan=10
- **Seasonality**: holiday_data_surge(Dec, +1.3x data), summer_roaming(Jun-Aug, +1.2x)

## Telecom > Churn Prediction
- **Fact**: `fact_churn_signal` (signal_id INT, subscriber_id INT FK→dim_subscriber, date_id INT FK→dim_date, usage_trend NUMBER(5,3), complaint_count INT, nps_score INT, churn_prob NUMBER(5,3))
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, tenure_months, plan_price, contract_end), `dim_segment`(segment_id, risk_label, value_tier)
- **Grain**: One row per subscriber per week
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: churn_prob=NORMAL(0.08,0.12), complaint_count=ZIPF(2.5), nps_score=NORMAL(35,18), subscriber_id=UNIFORM
- **Dim Sizes**: dim_subscriber=500, dim_segment=10
- **Seasonality**: contract_end_cluster(month 12/24, +2.0x churn), post_price_increase(event, +1.5x)

## Telecom > Network Performance
- **Fact**: `fact_cell_kpi` (kpi_id INT, tower_id INT FK→dim_tower, date_id INT FK→dim_date, throughput_mbps NUMBER(8,2), latency_ms NUMBER(6,2), drop_rate NUMBER(5,4), traffic_gb NUMBER(10,2))
- **Dims**: `dim_date`, `dim_tower`(tower_id, location, type, capacity, vendor), `dim_region`(region_id, name, population_density)
- **Grain**: One row per tower per hour
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: throughput_mbps=NORMAL(85,30), latency_ms=NORMAL(22,10), tower_id=UNIFORM, traffic_gb=ZIPF(1.7)
- **Dim Sizes**: dim_tower=200, dim_region=20
- **Seasonality**: event_surge(concerts/sports, +3.0x local), evening_peak(18-22h, +1.8x daily), new_year_eve(Dec 31, +5.0x SMS)

## Telecom > Revenue Assurance
- **Fact**: `fact_cdr_reconciliation` (recon_id INT, subscriber_id INT FK→dim_subscriber, date_id INT FK→dim_date, rated_amount NUMBER(8,4), billed_amount NUMBER(8,4), variance NUMBER(8,4), leakage_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, plan_id, roaming_flag), `dim_service_type`(service_type_id, name, rate_per_unit)
- **Grain**: One row per CDR rated event
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: rated_amount=NORMAL(0.15,0.3), variance=ZIPF(2.5), leakage_flag=UNIFORM(0.015), subscriber_id=ZIPF(1.1)
- **Dim Sizes**: dim_subscriber=500, dim_service_type=15
- **Seasonality**: roaming_season(Jun-Aug, +1.5x variance), system_upgrade(event, +2.0x leakage)
- **Note**: rated_amount=NORMAL(0.15,0.3) intentionally allows negative values (represents billing variance/adjustments between rated and billed amounts, not absolute amounts)

## Energy & Utilities > Grid Reliability
- **Fact**: `fact_outage_event` (outage_id INT, feeder_id INT FK→dim_feeder, date_id INT FK→dim_date, duration_min NUMBER(8,0), customers_affected INT, cause VARCHAR, crew_response_min INT)
- **Dims**: `dim_date`, `dim_feeder`(feeder_id, substation, voltage_class, circuit_miles, tree_density), `dim_cause`(cause_id, category, weather_related)
- **Grain**: One row per outage event
- **Volume**: ~20K rows/year at medium scale
- **Distributions**: duration_min=ZIPF(1.7), customers_affected=ZIPF(1.9), feeder_id=ZIPF(1.1)
- **Dim Sizes**: dim_feeder=100, dim_cause=20
- **Seasonality**: storm_season(Jun-Sep, +2.5x), ice_storms(Dec-Feb, +1.8x), calm_fall(Oct, 0.6x)

## Energy & Utilities > Demand Forecasting
- **Fact**: `fact_load_interval` (interval_id INT, meter_zone_id INT FK→dim_meter_zone, date_id INT FK→dim_date, hour INT, demand_mw NUMBER(10,3), temperature_f NUMBER(5,1), forecast_mw NUMBER(10,3))
- **Dims**: `dim_date`, `dim_meter_zone`(zone_id, region, customer_class, peak_capacity_mw), `dim_weather`(weather_id, temp, humidity, cloud_cover)
- **Grain**: One row per meter zone per hour
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: demand_mw=NORMAL(450,150), temperature_f=NORMAL(62,18), meter_zone_id=UNIFORM
- **Dim Sizes**: dim_meter_zone=50, dim_weather=365
- **Seasonality**: summer_peak(Jul-Aug, +1.6x AC load), winter_heat(Jan-Feb, +1.3x), shoulder_low(Apr+Oct, 0.75x)

## Energy & Utilities > Smart Meter Analytics
- **Fact**: `fact_meter_read` (read_id INT, meter_id INT FK→dim_meter, date_id INT FK→dim_date, kwh_consumed NUMBER(8,3), tamper_flag BOOLEAN, voltage_deviation NUMBER(5,2), read_quality VARCHAR)
- **Dims**: `dim_date`, `dim_meter`(meter_id, customer_id, meter_type, install_date, location), `dim_rate`(rate_id, tariff_code, tou_period)
- **Grain**: One row per meter per 15-minute interval (aggregated to daily for medium scale)
- **Volume**: ~500K rows/year at medium scale (daily grain)
- **Distributions**: kwh_consumed=NORMAL(28,15), tamper_flag=UNIFORM(0.005), meter_id=UNIFORM, voltage_deviation=NORMAL(0,2.5)
- **Dim Sizes**: dim_meter=1000, dim_rate=10
- **Seasonality**: summer_consumption(Jun-Aug, +1.5x), winter_heating(Dec-Feb, +1.3x), weekend_pattern(Sat-Sun, −0.2x commercial)

## Logistics & Transportation > Route Optimization
- **Fact**: `fact_delivery_stop` (stop_id INT, route_id INT FK→dim_route, driver_id INT FK→dim_driver, date_id INT FK→dim_date, stop_sequence INT, dwell_min NUMBER(6,1), distance_mi NUMBER(8,2), on_time BOOLEAN)
- **Dims**: `dim_date`, `dim_route`(route_id, origin, region, planned_stops), `dim_driver`(driver_id, name, vehicle_type, shift)
- **Grain**: One row per delivery stop
- **Volume**: ~350K rows/year at medium scale
- **Distributions**: dwell_min=NORMAL(8,5), distance_mi=NORMAL(4.5,3), route_id=ZIPF(1.1), on_time=UNIFORM(0.88)
- **Dim Sizes**: dim_route=200, dim_driver=100
- **Seasonality**: holiday_volume(Nov-Dec, +1.8x), summer_travel(Jun-Aug, +1.1x traffic), monday_surge(Mon, +1.2x)

## Logistics & Transportation > Fleet Utilization
- **Fact**: `fact_vehicle_day` (record_id INT, vehicle_id INT FK→dim_vehicle, date_id INT FK→dim_date, miles_driven NUMBER(8,1), fuel_gallons NUMBER(6,2), idle_pct NUMBER(5,2), utilization_pct NUMBER(5,2))
- **Dims**: `dim_date`, `dim_vehicle`(vehicle_id, type, capacity, model_year, home_base), `dim_driver`(driver_id, license_class, safety_score)
- **Grain**: One row per vehicle per day
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: miles_driven=NORMAL(180,80), fuel_gallons=NORMAL(25,12), vehicle_id=UNIFORM, utilization_pct=NORMAL(0.72,0.15)
- **Dim Sizes**: dim_vehicle=150, dim_driver=80
- **Seasonality**: peak_shipping(Oct-Dec, +1.4x utilization), maintenance_down(Jan, −0.2x fleet), weather_delays(winter, +0.15x idle)

## Logistics & Transportation > Shipment Visibility
- **Fact**: `fact_shipment_event` (event_id INT, shipment_id INT FK→dim_shipment, date_id INT FK→dim_date, milestone VARCHAR, location VARCHAR, delay_hours NUMBER(6,1), exception_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_shipment`(shipment_id, origin, destination, carrier_id, mode, weight_lbs), `dim_carrier`(carrier_id, name, type, rating)
- **Grain**: One row per shipment milestone event
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: delay_hours=ZIPF(2.0), shipment_id=UNIFORM, exception_flag=UNIFORM(0.08), carrier_id=ZIPF(1.1)
- **Dim Sizes**: dim_shipment=2000, dim_carrier=30
- **Seasonality**: peak_season(Oct-Dec, +1.6x volume), port_congestion(Jan-Feb, +1.5x delays), summer_smooth(Jun, 0.85x exceptions)

## Media & Entertainment > Content Recommendation
- **Fact**: `fact_view_event` (view_id INT, user_id INT FK→dim_user, content_id INT FK→dim_content, date_id INT FK→dim_date, watch_min NUMBER(6,1), completion_pct NUMBER(5,2), rating NUMBER(3,1))
- **Dims**: `dim_date`, `dim_user`(user_id, tier, signup_date, device_primary), `dim_content`(content_id, title, genre, release_year, duration_min)
- **Grain**: One row per viewing session
- **Volume**: ~800K rows/year at medium scale
- **Distributions**: watch_min=NORMAL(35,25), content_id=ZIPF(1.1), user_id=ZIPF(1.1), completion_pct=NORMAL(0.6,0.3)
- **Dim Sizes**: dim_user=1000, dim_content=500
- **Seasonality**: weekend_binge(Sat-Sun, +1.5x), winter_evenings(Nov-Feb, +1.3x), summer_outdoor(Jun-Jul, 0.8x)

## Media & Entertainment > Viewer Engagement
- **Fact**: `fact_session` (session_id INT, user_id INT FK→dim_user, date_id INT FK→dim_date, session_duration_sec INT, events_count INT, content_items INT, drop_off_point VARCHAR)
- **Dims**: `dim_date`, `dim_user`(user_id, cohort_month, plan, device), `dim_platform`(platform_id, name, os, app_version)
- **Grain**: One row per user session
- **Volume**: ~600K rows/year at medium scale
- **Distributions**: session_duration_sec=NORMAL(1800,1200), content_items=ZIPF(1.9), user_id=ZIPF(1.1), events_count=NORMAL(25,15)
- **Dim Sizes**: dim_user=800, dim_platform=10
- **Seasonality**: evening_peak(19-23h, +2.0x), release_day_spike(new content, +3.0x), summer_dip(Jun-Aug, 0.85x)

## Media & Entertainment > Ad Yield
- **Fact**: `fact_impression` (impression_id INT, placement_id INT FK→dim_placement, campaign_id INT FK→dim_campaign, date_id INT FK→dim_date, cpm NUMBER(8,4), bid_amount NUMBER(8,4), filled BOOLEAN, clicked BOOLEAN)
- **Dims**: `dim_date`, `dim_placement`(placement_id, slot_type, page_position, content_genre), `dim_campaign`(campaign_id, advertiser, budget, targeting_segment)
- **Grain**: One row per ad impression opportunity
- **Volume**: ~1000K rows/year at medium scale
- **Distributions**: cpm=NORMAL(12,8), bid_amount=ZIPF(1.8), campaign_id=ZIPF(1.1), filled=UNIFORM(0.85), clicked=UNIFORM(0.02)
- **Dim Sizes**: dim_placement=200, dim_campaign=500
- **Seasonality**: q4_ad_spend(Oct-Dec, +1.8x CPM), january_reset(Jan, 0.5x budgets), upfront_season(May-Jun, +1.2x)

## Media & Entertainment > Churn Prediction
- **Fact**: `fact_subscriber_signal` (signal_id INT, subscriber_id INT FK→dim_subscriber, date_id INT FK→dim_date, engagement_score NUMBER(5,2), days_since_login INT, billing_issue BOOLEAN, churn_prob NUMBER(5,3))
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, plan, tenure_months, acquisition_source), `dim_content_pref`(pref_id, top_genre, diversity_score)
- **Grain**: One row per subscriber per week
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: engagement_score=NORMAL(55,22), days_since_login=ZIPF(2.0), churn_prob=NORMAL(0.06,0.08), subscriber_id=UNIFORM
- **Dim Sizes**: dim_subscriber=500, dim_content_pref=100
- **Seasonality**: post_binge_churn(after major release +30d, +1.5x), price_increase(event, +2.0x), summer_cancel(Jun, +1.3x)

## Life Sciences & Pharma > Clinical Trial Optimization
- **Fact**: `fact_enrollment` (enrollment_id INT, site_id INT FK→dim_site, trial_id INT FK→dim_trial, date_id INT FK→dim_date, screened INT, enrolled INT, screen_fail_rate NUMBER(5,3), days_to_enroll INT)
- **Dims**: `dim_date`, `dim_site`(site_id, name, country, pi_name, capacity), `dim_trial`(trial_id, phase, therapeutic_area, target_n)
- **Grain**: One row per site per week
- **Volume**: ~50K rows/year at medium scale
- **Distributions**: screened=NORMAL(8,5), enrolled=NORMAL(3,2), site_id=ZIPF(1.1), screen_fail_rate=NORMAL(0.45,0.15)
- **Dim Sizes**: dim_site=100, dim_trial=20
- **Seasonality**: summer_slowdown(Jul-Aug, −0.3x EU sites), holiday_pause(Dec, −0.4x), q1_startup(Jan-Feb, +1.3x new sites)

## Life Sciences & Pharma > Pharmacovigilance
- **Fact**: `fact_adverse_event` (ae_id INT, drug_id INT FK→dim_drug, reporter_id INT FK→dim_reporter, date_id INT FK→dim_date, severity VARCHAR, seriousness BOOLEAN, outcome VARCHAR, days_to_onset INT)
- **Dims**: `dim_date`, `dim_drug`(drug_id, name, therapeutic_class, market_date), `dim_reporter`(reporter_id, type, country), `dim_meddra`(term_id, pt_name, soc, hlgt)
- **Grain**: One row per adverse event case
- **Volume**: ~30K rows/year at medium scale
- **Distributions**: days_to_onset=ZIPF(1.8), drug_id=ZIPF(1.1), seriousness=UNIFORM(0.35), reporter_id=ZIPF(1.1)
- **Dim Sizes**: dim_drug=50, dim_reporter=200, dim_meddra=500
- **Seasonality**: post_launch_spike(first 6 months, +3.0x), regulatory_deadline(quarterly, +1.5x reporting)

## Life Sciences & Pharma > RWE Analytics
- **Fact**: `fact_patient_outcome` (outcome_id INT, patient_id INT FK→dim_patient, drug_id INT FK→dim_drug, date_id INT FK→dim_date, treatment_duration_days INT, response BOOLEAN, cost_total NUMBER(12,2), er_visits INT)
- **Dims**: `dim_date`, `dim_patient`(patient_id, age_band, gender, comorbidity_count, payer_type), `dim_drug`(drug_id, name, class, route)
- **Grain**: One row per patient treatment episode
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: treatment_duration_days=NORMAL(180,90), cost_total=ZIPF(1.7), er_visits=ZIPF(2.5), patient_id=ZIPF(1.1)
- **Dim Sizes**: dim_patient=500, dim_drug=30
- **Seasonality**: flu_comorbidity(Nov-Feb, +1.2x ER), post_diagnosis_wave(continuous)

## Life Sciences & Pharma > Commercial Analytics
- **Fact**: `fact_rx_sales` (rx_id INT, hcp_id INT FK→dim_hcp, territory_id INT FK→dim_territory, date_id INT FK→dim_date, trx INT, nrx INT, market_share NUMBER(5,3), rep_calls INT)
- **Dims**: `dim_date`, `dim_hcp`(hcp_id, npi, specialty, decile, state), `dim_territory`(territory_id, rep_id, region, target_hcp_count)
- **Grain**: One row per HCP per week
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: trx=ZIPF(1.8), nrx=ZIPF(2.0), hcp_id=ZIPF(1.1), market_share=NORMAL(0.15,0.08)
- **Dim Sizes**: dim_hcp=500, dim_territory=50
- **Seasonality**: flu_rx(Oct-Feb, +1.4x respiratory), launch_curve(months 1-12, logarithmic growth), summer_dip(Jul, 0.85x)

## Public Sector > Constituent Service
- **Fact**: `fact_service_request` (request_id INT, constituent_id INT FK→dim_constituent, department_id INT FK→dim_department, date_id INT FK→dim_date, resolution_hours NUMBER(8,1), priority INT, channel VARCHAR, satisfied BOOLEAN)
- **Dims**: `dim_date`, `dim_constituent`(constituent_id, district, zip, language), `dim_department`(dept_id, name, category, sla_hours)
- **Grain**: One row per service request
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: resolution_hours=ZIPF(1.8), constituent_id=ZIPF(1.1), department_id=ZIPF(1.1), priority=UNIFORM(1-5)
- **Dim Sizes**: dim_constituent=1000, dim_department=20
- **Seasonality**: pothole_spring(Mar-Apr, +1.5x infrastructure), summer_parks(Jun-Aug, +1.3x), snow_winter(Dec-Feb, +1.4x)

## Public Sector > Fraud/Waste/Abuse
- **Fact**: `fact_benefit_payment` (payment_id INT, recipient_id INT FK→dim_recipient, program_id INT FK→dim_program, date_id INT FK→dim_date, amount NUMBER(10,2), fraud_flag BOOLEAN, overpayment NUMBER(10,2), detection_method VARCHAR)
- **Dims**: `dim_date`, `dim_recipient`(recipient_id, eligibility_status, income_band, household_size), `dim_program`(program_id, name, type, annual_budget)
- **Grain**: One row per benefit payment
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: amount=NORMAL(1200,800), fraud_flag=UNIFORM(0.04), overpayment=ZIPF(2.0), recipient_id=ZIPF(1.1)
- **Dim Sizes**: dim_recipient=2000, dim_program=15
- **Seasonality**: year_end_rush(Nov-Dec, +1.3x), enrollment_period(Oct, +1.5x), tax_season_fraud(Jan-Mar, +1.4x fraud)

## Public Sector > Public Safety
- **Fact**: `fact_incident` (incident_id INT, location_id INT FK→dim_location, officer_id INT FK→dim_officer, date_id INT FK→dim_date, response_min NUMBER(6,1), severity INT, incident_type VARCHAR, arrests INT)
- **Dims**: `dim_date`, `dim_location`(location_id, district, beat, lat, lon, land_use), `dim_officer`(officer_id, unit, rank, shift)
- **Grain**: One row per reported incident
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: response_min=NORMAL(8,5), location_id=ZIPF(1.1), severity=ZIPF(2.5), officer_id=ZIPF(1.1)
- **Dim Sizes**: dim_location=300, dim_officer=200
- **Seasonality**: summer_crime(Jun-Aug, +1.4x), weekend_spike(Fri-Sat, +1.6x), holiday_dv(Dec, +1.2x domestic)

## Healthcare > Clinical Trial Analytics
- **Fact**: `fact_trial_patient` (trial_patient_id INT, patient_id INT FK→dim_patient, site_id INT FK→dim_site, protocol_id INT FK→dim_protocol, date_id INT FK→dim_date, visit_adherence_pct NUMBER(5,2), adverse_events INT, lab_abnormal_flag BOOLEAN, dropout BOOLEAN)
- **Dims**: `dim_date`, `dim_patient`(patient_id, age_band, gender, comorbidity_count, eligibility_criteria_met), `dim_site`(site_id, name, country, pi_name, capacity), `dim_protocol`(protocol_id, phase, therapeutic_area, target_n, duration_weeks)
- **Grain**: One row per patient per protocol visit
- **Volume**: ~60K rows/year at medium scale
- **Distributions**: visit_adherence_pct=NORMAL(0.85,0.12), adverse_events=ZIPF(2.5), patient_id=UNIFORM, site_id=ZIPF(1.1), dropout=UNIFORM(0.18)
- **Dim Sizes**: dim_patient=500, dim_site=50, dim_protocol=15
- **Seasonality**: summer_slowdown(Jul-Aug, −0.3x enrollment), holiday_pause(Dec, −0.4x), q1_startup(Jan-Feb, +1.3x)

## Healthcare > Revenue Cycle Optimization
- **Fact**: `fact_claim_lifecycle` (claim_lifecycle_id INT, encounter_id INT FK→dim_encounter, payer_id INT FK→dim_payer, date_id INT FK→dim_date, charge_amount NUMBER(12,2), paid_amount NUMBER(12,2), denial_flag BOOLEAN, days_to_payment INT, denial_reason VARCHAR)
- **Dims**: `dim_date`, `dim_encounter`(encounter_id, service_type, facility_id, provider_id), `dim_payer`(payer_id, name, type, plan_category)
- **Grain**: One row per claim submission event
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: charge_amount=NORMAL(1800,2500), paid_amount=NORMAL(1200,1800), denial_flag=UNIFORM(0.12), days_to_payment=NORMAL(35,20), payer_id=ZIPF(1.1)
- **Dim Sizes**: dim_encounter=5000, dim_payer=50
- **Seasonality**: year_end_rush(Dec, +1.3x submissions), q1_resubmit(Jan-Feb, +1.2x denials), summer_dip(Jul, 0.9x)

## Healthcare > Medication Adherence & Pharmacy Analytics
- **Fact**: `fact_fill_event` (fill_id INT, patient_id INT FK→dim_patient, medication_id INT FK→dim_medication, date_id INT FK→dim_date, days_supply INT, gap_days INT, pdc_ratio NUMBER(5,3), refill_on_time BOOLEAN)
- **Dims**: `dim_date`, `dim_patient`(patient_id, age_band, chronic_conditions, payer_type, zip), `dim_medication`(medication_id, ndc, generic_name, therapeutic_class, route)
- **Grain**: One row per prescription fill
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: days_supply=NORMAL(30,10), gap_days=ZIPF(2.0), pdc_ratio=NORMAL(0.72,0.2), patient_id=ZIPF(1.1), refill_on_time=UNIFORM(0.65)
- **Dim Sizes**: dim_patient=800, dim_medication=200
- **Seasonality**: q1_new_year_adherence(Jan, +1.2x fills), summer_gap(Jun-Aug, +1.3x gap_days), flu_season_rx(Oct-Feb, +1.2x respiratory Rx)

## Healthcare > Provider Performance & Network Adequacy
- **Fact**: `fact_provider_metric` (metric_id INT, provider_id INT FK→dim_provider, measure_id INT FK→dim_quality_measure, date_id INT FK→dim_date, score NUMBER(5,2), patient_panel_size INT, cost_per_episode NUMBER(10,2), readmit_rate NUMBER(5,3))
- **Dims**: `dim_date`, `dim_provider`(provider_id, npi, specialty, facility_type, network_status, region), `dim_quality_measure`(measure_id, name, category, benchmark)
- **Grain**: One row per provider per quality measure per quarter
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: score=NORMAL(75,15), patient_panel_size=NORMAL(350,150), cost_per_episode=NORMAL(4500,2500), provider_id=UNIFORM, readmit_rate=NORMAL(0.12,0.05)
- **Dim Sizes**: dim_provider=200, dim_quality_measure=40
- **Seasonality**: quarterly_reporting(Mar/Jun/Sep/Dec, +2.0x submissions), year_end_push(Dec, +1.3x)

## Healthcare > Social Determinants of Health (SDOH)
- **Fact**: `fact_sdoh_assessment` (assessment_id INT, patient_id INT FK→dim_patient, community_id INT FK→dim_community, date_id INT FK→dim_date, food_insecurity_flag BOOLEAN, housing_instability_flag BOOLEAN, transportation_barrier_flag BOOLEAN, sdoh_risk_score NUMBER(5,2), referral_count INT)
- **Dims**: `dim_date`, `dim_patient`(patient_id, age_band, gender, zip, payer_type, chronic_count), `dim_community`(community_id, census_tract, adi_rank, food_desert_flag, median_income)
- **Grain**: One row per patient SDOH assessment
- **Volume**: ~50K rows/year at medium scale
- **Distributions**: food_insecurity_flag=UNIFORM(0.12), housing_instability_flag=UNIFORM(0.08), transportation_barrier_flag=UNIFORM(0.15), sdoh_risk_score=NORMAL(35,20), patient_id=ZIPF(1.1), referral_count=ZIPF(2.2)
- **Dim Sizes**: dim_patient=600, dim_community=100
- **Seasonality**: winter_housing(Dec-Feb, +1.4x housing instability), summer_food(Jun-Aug, +1.2x food insecurity—school meals end), year_end_review(Dec, +1.3x)

## Healthcare > Real-Time Clinical Decision Support
- **Fact**: `fact_clinical_alert` (alert_id INT, patient_id INT FK→dim_patient, rule_id INT FK→dim_alert_rule, date_id INT FK→dim_date, alert_type VARCHAR, severity INT, acknowledged BOOLEAN, action_taken VARCHAR, response_min NUMBER(6,1))
- **Dims**: `dim_date`, `dim_patient`(patient_id, age, acuity_level, location, attending_id), `dim_alert_rule`(rule_id, name, category, threshold, evidence_level)
- **Grain**: One row per clinical alert fired
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: severity=ZIPF(2.0), response_min=NORMAL(12,8), patient_id=ZIPF(1.1), rule_id=ZIPF(1.1), acknowledged=UNIFORM(0.78)
- **Dim Sizes**: dim_patient=500, dim_alert_rule=80
- **Seasonality**: flu_season_alerts(Nov-Feb, +1.4x), night_shift_spike(overnight, +1.2x alert fatigue), monday_surge(Mon, +1.1x)
- **Intraday**: Pattern #13 hospital 24h (peaks 8-10h, 14-16h, lower overnight)

## Retail & CPG > Supply Chain Visibility & Optimization
- **Fact**: `fact_po_shipment` (shipment_id INT, supplier_id INT FK→dim_supplier, product_id INT FK→dim_product, date_id INT FK→dim_date, order_qty INT, received_qty INT, lead_days INT, unit_cost NUMBER(8,4), on_time BOOLEAN)
- **Dims**: `dim_date`, `dim_supplier`(supplier_id, name, country, tier, reliability_score), `dim_product`(product_id, sku, category, brand, shelf_life_days)
- **Grain**: One row per purchase order delivery receipt
- **Volume**: ~120K rows/year at medium scale
- **Distributions**: lead_days=NORMAL(12,6), unit_cost=ZIPF(1.6), supplier_id=ZIPF(1.1), on_time=UNIFORM(0.82), order_qty=NORMAL(500,300)
- **Dim Sizes**: dim_supplier=150, dim_product=400
- **Seasonality**: holiday_stockpile(Sep-Oct, +1.5x), chinese_new_year_delay(Feb, +1.8x lead_days), post_holiday_lull(Jan, 0.7x)

## Retail & CPG > Promotion Effectiveness & Marketing Attribution
- **Fact**: `fact_promo_response` (response_id INT, campaign_id INT FK→dim_campaign, customer_id INT FK→dim_customer, date_id INT FK→dim_date, channel VARCHAR, impressions INT, conversions INT, revenue_attributed NUMBER(10,2), roi NUMBER(5,2))
- **Dims**: `dim_date`, `dim_campaign`(campaign_id, name, type, budget, start_date, end_date), `dim_customer`(customer_id, segment, loyalty_tier, zip)
- **Grain**: One row per campaign per customer touchpoint
- **Volume**: ~600K rows/year at medium scale
- **Distributions**: impressions=ZIPF(1.8), conversions=ZIPF(2.5), revenue_attributed=ZIPF(1.9), customer_id=ZIPF(1.1), campaign_id=ZIPF(1.1)
- **Dim Sizes**: dim_campaign=100, dim_customer=1000
- **Seasonality**: holiday_campaigns(Nov-Dec, +2.0x), back_to_school(Aug, +1.3x), post_holiday_clearance(Jan, +1.5x)

## Retail & CPG > Store Performance & Location Intelligence
- **Fact**: `fact_store_daily` (store_daily_id INT, store_id INT FK→dim_store, date_id INT FK→dim_date, revenue NUMBER(10,2), transactions INT, traffic_count INT, conversion_rate NUMBER(5,3), avg_basket NUMBER(8,2))
- **Dims**: `dim_date`, `dim_store`(store_id, name, region, format, sqft, lat, lon, trade_area_pop)
- **Grain**: One row per store per day
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: revenue=NORMAL(25000,12000), transactions=NORMAL(450,200), traffic_count=NORMAL(800,350), store_id=UNIFORM, conversion_rate=NORMAL(0.55,0.1)
- **Dim Sizes**: dim_store=50
- **Seasonality**: holiday_peak(Nov-Dec, +1.6x), summer_weekend(Jun-Aug Sat-Sun, +1.2x), monday_dip(Mon, 0.8x)
- **Intraday**: Pattern #13 retail bimodal (peaks 12-14h, 17-19h)

## Retail & CPG > Assortment Optimization
- **Fact**: `fact_sku_performance` (perf_id INT, product_id INT FK→dim_product, store_cluster_id INT FK→dim_store_cluster, date_id INT FK→dim_date, units_sold INT, revenue NUMBER(10,2), margin_pct NUMBER(5,3), stockout_days INT, shelf_space_pct NUMBER(5,3))
- **Dims**: `dim_date`, `dim_product`(product_id, sku, category, subcategory, brand, size_variant), `dim_store_cluster`(cluster_id, name, store_count, demo_profile, region)
- **Grain**: One row per product per store cluster per week
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: units_sold=NORMAL(25,18), revenue=NORMAL(80,60), margin_pct=NORMAL(0.32,0.12), product_id=ZIPF(1.1), stockout_days=ZIPF(2.5)
- **Dim Sizes**: dim_product=500, dim_store_cluster=25
- **Seasonality**: new_planogram(Jan+Jul, +1.3x SKU changes), seasonal_rotation(Mar+Sep, +1.2x)

## Retail & CPG > Returns & Fraud Analytics
- **Fact**: `fact_return` (return_id INT, customer_id INT FK→dim_customer, product_id INT FK→dim_product, date_id INT FK→dim_date, return_amount NUMBER(10,2), reason_code VARCHAR, channel VARCHAR, fraud_score NUMBER(5,3), receipt_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_customer`(customer_id, loyalty_tier, return_history_count, signup_date), `dim_product`(product_id, category, brand, unit_price)
- **Grain**: One row per return transaction
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: return_amount=NORMAL(55,40), fraud_score=NORMAL(0.08,0.12), customer_id=ZIPF(1.1), product_id=ZIPF(1.1), receipt_flag=UNIFORM(0.72)
- **Dim Sizes**: dim_customer=800, dim_product=400
- **Seasonality**: post_holiday_returns(Jan, +2.5x), post_black_friday(Dec first week, +1.8x), summer_low(Jul, 0.7x)

## Retail & CPG > Sustainability & Waste Reduction
- **Fact**: `fact_waste_event` (waste_id INT, product_id INT FK→dim_product, store_id INT FK→dim_store, date_id INT FK→dim_date, waste_qty NUMBER(8,2), waste_value NUMBER(8,2), waste_type VARCHAR, days_before_expiry INT, donated_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_product`(product_id, category, shelf_life_days, organic_flag, perishable_flag), `dim_store`(store_id, region, format, waste_diversion_rate)
- **Grain**: One row per waste/shrink event
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: waste_qty=ZIPF(2.0), waste_value=NORMAL(12,10), product_id=ZIPF(1.1), store_id=ZIPF(1.1), days_before_expiry=NORMAL(2,3), donated_flag=UNIFORM(0.15)
- **Dim Sizes**: dim_product=400, dim_store=50
- **Seasonality**: summer_spoilage(Jun-Aug, +1.5x perishable), holiday_overstock(Jan, +1.4x), produce_season(Apr-Jun, +1.2x)

## Financial Services > Market Risk & Portfolio Analytics
- **Fact**: `fact_position_snapshot` (snapshot_id INT, instrument_id INT FK→dim_instrument, portfolio_id INT FK→dim_portfolio, date_id INT FK→dim_date, market_value NUMBER(14,2), var_1d NUMBER(12,2), delta NUMBER(10,4), pnl_daily NUMBER(12,2))
- **Dims**: `dim_date`, `dim_instrument`(instrument_id, cusip, asset_class, issuer, currency, maturity_date), `dim_portfolio`(portfolio_id, name, strategy, desk, manager)
- **Grain**: One row per instrument per portfolio per day
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: market_value=ZIPF(1.8), var_1d=NORMAL(50000,80000), pnl_daily=NORMAL(0,25000), instrument_id=ZIPF(1.1), portfolio_id=ZIPF(1.1)
- **Dim Sizes**: dim_instrument=500, dim_portfolio=30
- **Seasonality**: quarter_end_rebalance(Mar/Jun/Sep/Dec, +1.3x trades), year_end_window_dressing(Dec, +1.2x), summer_low_vol(Jul-Aug, 0.85x)

## Financial Services > Regulatory Reporting (Basel/IFRS/CECL)
- **Fact**: `fact_regulatory_exposure` (exposure_id INT, account_id INT FK→dim_account, date_id INT FK→dim_date, exposure_at_default NUMBER(14,2), risk_weight NUMBER(5,3), ecl_provision NUMBER(12,2), stage INT, pd NUMBER(5,4), lgd NUMBER(5,3))
- **Dims**: `dim_date`, `dim_account`(account_id, product_type, customer_segment, collateral_type, origination_date), `dim_regulatory_class`(class_id, name, framework, asset_class)
- **Grain**: One row per account per reporting period (monthly)
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: exposure_at_default=ZIPF(1.8), ecl_provision=ZIPF(2.0), pd=NORMAL(0.02,0.03), lgd=NORMAL(0.4,0.15), account_id=UNIFORM
- **Dim Sizes**: dim_account=1000, dim_regulatory_class=20
- **Seasonality**: quarter_end_filing(Mar/Jun/Sep/Dec, +1.5x), year_end_recalibration(Dec, +1.3x model updates)

## Financial Services > Real-Time Payments & Settlement
- **Fact**: `fact_payment_event` (payment_id INT, account_id INT FK→dim_account, counterparty_id INT FK→dim_counterparty, date_id INT FK→dim_date, amount NUMBER(14,2), status VARCHAR, settlement_lag_sec INT, payment_rail VARCHAR, rejection_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_account`(account_id, account_type, currency, branch_id), `dim_counterparty`(counterparty_id, name, bank_code, country, risk_tier)
- **Grain**: One row per payment instruction
- **Volume**: ~600K rows/year at medium scale
- **Distributions**: amount=ZIPF(1.9), settlement_lag_sec=NORMAL(5,12), account_id=ZIPF(1.1), counterparty_id=ZIPF(1.1), rejection_flag=UNIFORM(0.02)
- **Dim Sizes**: dim_account=800, dim_counterparty=300
- **Seasonality**: month_end_settlement(last 2 days, +2.5x), tax_season(Mar-Apr, +1.4x), year_end(Dec, +1.3x)
- **Intraday**: Pattern #13 business hours (peak 9-16h, near-zero overnight)

## Financial Services > Personalized Financial Advice / Next-Best-Action
- **Fact**: `fact_recommendation_event` (rec_id INT, customer_id INT FK→dim_customer, product_id INT FK→dim_product, date_id INT FK→dim_date, channel VARCHAR, propensity_score NUMBER(5,3), presented BOOLEAN, accepted BOOLEAN, revenue_potential NUMBER(10,2))
- **Dims**: `dim_date`, `dim_customer`(customer_id, segment, aum_band, life_stage, relationship_tenure_months), `dim_product`(product_id, name, category, min_balance, fee_pct)
- **Grain**: One row per recommendation presented to customer
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: propensity_score=NORMAL(0.25,0.15), revenue_potential=ZIPF(1.9), customer_id=ZIPF(1.1), product_id=ZIPF(1.1), accepted=UNIFORM(0.08)
- **Dim Sizes**: dim_customer=500, dim_product=30
- **Seasonality**: new_year_planning(Jan, +1.5x), tax_season_investment(Mar-Apr, +1.3x), year_end_rebalance(Nov-Dec, +1.2x)

## Financial Services > Operational Risk & Loss Event Analysis
- **Fact**: `fact_loss_event` (event_id INT, business_line_id INT FK→dim_business_line, risk_category_id INT FK→dim_risk_category, date_id INT FK→dim_date, loss_amount NUMBER(12,2), recovery_amount NUMBER(12,2), root_cause VARCHAR, control_failure_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_business_line`(bl_id, name, region, product_group), `dim_risk_category`(category_id, name, basel_event_type, level_1, level_2)
- **Grain**: One row per operational loss event
- **Volume**: ~15K rows/year at medium scale
- **Distributions**: loss_amount=ZIPF(1.9), recovery_amount=ZIPF(2.2), business_line_id=ZIPF(1.1), risk_category_id=ZIPF(1.1), control_failure_flag=UNIFORM(0.35)
- **Dim Sizes**: dim_business_line=20, dim_risk_category=30
- **Seasonality**: quarter_end_discovery(Mar/Jun/Sep/Dec, +1.4x reported), system_change_spike(post-release, +1.5x), year_end_review(Dec, +1.2x)

## Financial Services > ESG / Sustainable Finance Analytics
- **Fact**: `fact_esg_score` (score_id INT, company_id INT FK→dim_company, date_id INT FK→dim_date, esg_composite NUMBER(5,2), environmental_score NUMBER(5,2), social_score NUMBER(5,2), governance_score NUMBER(5,2), carbon_intensity NUMBER(10,2), taxonomy_aligned_pct NUMBER(5,3))
- **Dims**: `dim_date`, `dim_company`(company_id, name, sector, country, market_cap_band), `dim_framework`(framework_id, name, version, jurisdiction)
- **Grain**: One row per company per quarter
- **Volume**: ~40K rows/year at medium scale
- **Distributions**: esg_composite=NORMAL(55,18), carbon_intensity=ZIPF(1.7), taxonomy_aligned_pct=NORMAL(0.3,0.2), company_id=UNIFORM
- **Dim Sizes**: dim_company=500, dim_framework=10
- **Seasonality**: annual_disclosure(Mar-Apr, +2.0x updates), cop_season(Nov, +1.3x attention), regulation_effective(Jan, +1.5x new mandates)

## Insurance > Customer 360 & Retention
- **Fact**: `fact_policy_interaction` (interaction_id INT, customer_id INT FK→dim_customer, policy_id INT FK→dim_policy, date_id INT FK→dim_date, channel VARCHAR, interaction_type VARCHAR, nps_score INT, cross_sell_flag BOOLEAN, lapse_risk NUMBER(5,3))
- **Dims**: `dim_date`, `dim_customer`(customer_id, household_id, tenure_years, multi_policy_flag, state), `dim_policy`(policy_id, lob, status, premium_band, agent_id)
- **Grain**: One row per customer interaction or policy event
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: nps_score=NORMAL(35,22), lapse_risk=NORMAL(0.12,0.1), customer_id=ZIPF(1.1), policy_id=ZIPF(1.1), cross_sell_flag=UNIFORM(0.06)
- **Dim Sizes**: dim_customer=800, dim_policy=1200
- **Seasonality**: renewal_months(policy anniversary, +1.5x), year_end_review(Dec, +1.2x), rate_increase_churn(event-driven, +1.8x lapse risk)
- **Correlated FK**: policy_id → dim_policy → customer_id (derive customer from policy, multi-policy households share customer_id)

## Insurance > Reserve Adequacy & Loss Development
- **Fact**: `fact_reserve_movement` (movement_id INT, claim_id INT FK→dim_claim, date_id INT FK→dim_date, case_reserve NUMBER(12,2), paid_cumulative NUMBER(12,2), incurred_cumulative NUMBER(12,2), development_month INT, ibnr_estimate NUMBER(12,2))
- **Dims**: `dim_date`, `dim_claim`(claim_id, lob, accident_year, report_year, status, claimant_id), `dim_lob`(lob_id, name, long_tail_flag, avg_development_months)
- **Grain**: One row per claim per evaluation month
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: case_reserve=ZIPF(1.7), paid_cumulative=ZIPF(1.8), incurred_cumulative=ZIPF(1.7), claim_id=ZIPF(1.1), ibnr_estimate=ZIPF(2.0)
- **Dim Sizes**: dim_claim=2000, dim_lob=8
- **Seasonality**: quarter_close(Mar/Jun/Sep/Dec, +1.5x reviews), year_end_strengthening(Dec, +1.3x reserve increases), cat_development(hurricane season +3-6mo, +1.8x)

## Insurance > Agent/Broker Performance & Distribution
- **Fact**: `fact_producer_metric` (metric_id INT, producer_id INT FK→dim_producer, date_id INT FK→dim_date, new_policies INT, renewal_policies INT, written_premium NUMBER(12,2), loss_ratio NUMBER(5,3), retention_rate NUMBER(5,3), commission_earned NUMBER(10,2))
- **Dims**: `dim_date`, `dim_producer`(producer_id, name, type, state, appointment_date, tier), `dim_territory`(territory_id, region, market_size, competition_index)
- **Grain**: One row per producer per month
- **Volume**: ~60K rows/year at medium scale
- **Distributions**: written_premium=ZIPF(1.8), new_policies=NORMAL(12,8), loss_ratio=NORMAL(0.62,0.18), producer_id=UNIFORM, retention_rate=NORMAL(0.85,0.08)
- **Dim Sizes**: dim_producer=200, dim_territory=40
- **Seasonality**: renewal_cycle(Jan+Jul, +1.5x), year_end_binding(Dec, +1.4x new), summer_lull(Jun-Jul, 0.85x)

## Insurance > Regulatory Reporting & Compliance
- **Fact**: `fact_statutory_entry` (entry_id INT, line_id INT FK→dim_statutory_line, jurisdiction_id INT FK→dim_jurisdiction, date_id INT FK→dim_date, earned_premium NUMBER(12,2), incurred_loss NUMBER(12,2), expense_amount NUMBER(10,2), combined_ratio NUMBER(5,3))
- **Dims**: `dim_date`, `dim_statutory_line`(line_id, schedule, line_of_business, annual_statement_line), `dim_jurisdiction`(jurisdiction_id, state, filing_deadline, regulator)
- **Grain**: One row per statutory line per jurisdiction per quarter
- **Volume**: ~20K rows/year at medium scale
- **Distributions**: earned_premium=ZIPF(1.7), incurred_loss=ZIPF(1.8), expense_amount=NORMAL(500000,300000), combined_ratio=NORMAL(0.98,0.12)
- **Dim Sizes**: dim_statutory_line=50, dim_jurisdiction=55
- **Seasonality**: annual_filing(Mar, +3.0x), quarterly_filing(Jun/Sep/Dec, +2.0x), rate_filing_season(varies, +1.3x)

## Insurance > Subrogation & Recovery Optimization
- **Fact**: `fact_recovery_event` (recovery_id INT, claim_id INT FK→dim_claim, date_id INT FK→dim_date, demand_amount NUMBER(12,2), recovered_amount NUMBER(12,2), recovery_method VARCHAR, days_to_recover INT, status VARCHAR)
- **Dims**: `dim_date`, `dim_claim`(claim_id, lob, paid_amount, at_fault_party_flag, state), `dim_vendor`(vendor_id, name, type, success_rate)
- **Grain**: One row per recovery action per claim
- **Volume**: ~40K rows/year at medium scale
- **Distributions**: demand_amount=ZIPF(1.8), recovered_amount=ZIPF(1.9), days_to_recover=NORMAL(120,80), claim_id=ZIPF(1.1)
- **Dim Sizes**: dim_claim=1500, dim_vendor=20
- **Seasonality**: q1_prior_year_claims(Jan-Mar, +1.4x), statute_deadline_push(varies, +1.5x), year_end_close(Dec, +1.3x)

## Insurance > Parametric & Usage-Based Insurance
- **Fact**: `fact_telemetry_score` (score_id INT, policy_id INT FK→dim_policy, date_id INT FK→dim_date, driving_score NUMBER(5,2), miles_driven NUMBER(8,1), hard_brakes INT, night_driving_pct NUMBER(5,3), premium_adjustment_pct NUMBER(5,3), trigger_event_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_policy`(policy_id, lob, ubi_enrolled_flag, base_premium, vehicle_type), `dim_trigger`(trigger_id, peril, threshold_value, payout_formula)
- **Grain**: One row per policy per scoring period (weekly)
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: driving_score=NORMAL(72,15), miles_driven=NORMAL(220,120), hard_brakes=ZIPF(2.5), premium_adjustment_pct=NORMAL(0,0.08), trigger_event_flag=UNIFORM(0.005)
- **Dim Sizes**: dim_policy=1000, dim_trigger=15
- **Seasonality**: summer_driving(Jun-Aug, +1.3x miles), winter_hard_brakes(Dec-Feb, +1.4x), holiday_travel(Dec, +1.2x)

## Technology/SaaS > Feature Adoption & Engagement
- **Fact**: `fact_feature_usage` (usage_id INT, user_id INT FK→dim_user, feature_id INT FK→dim_feature, date_id INT FK→dim_date, usage_count INT, time_spent_sec INT, depth_score NUMBER(5,2), adopted_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_user`(user_id, account_id, role, plan_tier, signup_date), `dim_feature`(feature_id, name, category, release_date, tier_required, complexity)
- **Grain**: One row per user per feature per day
- **Volume**: ~600K rows/year at medium scale
- **Distributions**: usage_count=ZIPF(1.8), time_spent_sec=NORMAL(180,120), user_id=ZIPF(1.1), feature_id=ZIPF(1.1), depth_score=NORMAL(0.4,0.25), adopted_flag=UNIFORM(0.35)
- **Dim Sizes**: dim_user=1000, dim_feature=50
- **Seasonality**: post_release_spike(+14d after release, +2.5x), q1_engagement(Jan, +1.3x), summer_dip(Jul-Aug, 0.85x)

## Technology/SaaS > Infrastructure Cost Attribution
- **Fact**: `fact_resource_usage` (usage_id INT, tenant_id INT FK→dim_tenant, service_id INT FK→dim_service, date_id INT FK→dim_date, cpu_hours NUMBER(10,2), memory_gb_hours NUMBER(10,2), storage_gb NUMBER(10,2), egress_gb NUMBER(8,2), cost_usd NUMBER(10,4))
- **Dims**: `dim_date`, `dim_tenant`(tenant_id, name, plan_tier, arr_band, region), `dim_service`(service_id, name, component, cost_center, unit_rate)
- **Grain**: One row per tenant per service per day
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: cpu_hours=ZIPF(1.7), memory_gb_hours=ZIPF(1.6), storage_gb=NORMAL(80,100), cost_usd=ZIPF(1.8), tenant_id=ZIPF(1.1)
- **Dim Sizes**: dim_tenant=200, dim_service=25
- **Seasonality**: month_end_batch(last 3 days, +1.8x), q4_spike(Nov-Dec, +1.3x), weekend_low(Sat-Sun, 0.6x)

## Technology/SaaS > Support Ticket & Escalation Prediction
- **Fact**: `fact_ticket` (ticket_id INT, account_id INT FK→dim_account, product_area_id INT FK→dim_product_area, date_id INT FK→dim_date, priority INT, resolution_hours NUMBER(8,1), escalated BOOLEAN, csat_score INT, first_response_min INT)
- **Dims**: `dim_date`, `dim_account`(account_id, tier, arr_band, health_score, csm_id), `dim_product_area`(area_id, name, team, complexity_tier)
- **Grain**: One row per support ticket
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: resolution_hours=ZIPF(1.7), priority=ZIPF(2.0), account_id=ZIPF(1.1), escalated=UNIFORM(0.12), csat_score=NORMAL(4.0,0.8), first_response_min=NORMAL(45,30)
- **Dim Sizes**: dim_account=500, dim_product_area=15
- **Seasonality**: post_release_tickets(+7d after deploy, +2.0x), monday_surge(Mon, +1.3x), holiday_skeleton(Dec, 0.7x)

## Technology/SaaS > Multi-Tenant Performance Analytics
- **Fact**: `fact_perf_metric` (metric_id INT, tenant_id INT FK→dim_tenant, endpoint_id INT FK→dim_endpoint, date_id INT FK→dim_date, p50_latency_ms NUMBER(8,2), p99_latency_ms NUMBER(8,2), error_rate NUMBER(5,4), request_count INT, cpu_pct NUMBER(5,2))
- **Dims**: `dim_date`, `dim_tenant`(tenant_id, plan_tier, sla_target_ms, region), `dim_endpoint`(endpoint_id, path, service, method, criticality)
- **Grain**: One row per tenant per endpoint per hour
- **Volume**: ~800K rows/year at medium scale
- **Distributions**: p50_latency_ms=NORMAL(45,25), p99_latency_ms=NORMAL(250,150), error_rate=ZIPF(2.5), request_count=ZIPF(1.7), tenant_id=ZIPF(1.1)
- **Dim Sizes**: dim_tenant=200, dim_endpoint=40
- **Seasonality**: business_hours_peak(9-17h Mon-Fri, +1.8x), month_end(last 3 days, +1.4x), deploy_window(Thu, +1.2x errors)
- **Intraday**: Pattern #13 business hours (peak 10-16h, low overnight)

## Technology/SaaS > Partner & Marketplace Ecosystem Analytics
- **Fact**: `fact_integration_event` (event_id INT, partner_id INT FK→dim_partner, account_id INT FK→dim_account, date_id INT FK→dim_date, event_type VARCHAR, api_calls INT, revenue_share NUMBER(8,2), install_flag BOOLEAN, uninstall_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_partner`(partner_id, name, tier, category, integration_type, revenue_share_pct), `dim_account`(account_id, segment, plan_tier, integrations_count)
- **Grain**: One row per partner-account interaction event per day
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: api_calls=ZIPF(1.8), revenue_share=ZIPF(2.0), partner_id=ZIPF(1.1), account_id=ZIPF(1.1), install_flag=UNIFORM(0.02), uninstall_flag=UNIFORM(0.005)
- **Dim Sizes**: dim_partner=50, dim_account=500
- **Seasonality**: q4_marketplace_push(Oct-Dec, +1.4x installs), conference_bump(event-driven, +2.0x), new_year_cleanup(Jan, +1.5x uninstalls)

## Manufacturing > Production Planning & Scheduling
- **Fact**: `fact_work_order` (wo_id INT, machine_id INT FK→dim_machine, product_id INT FK→dim_product, date_id INT FK→dim_date, planned_qty INT, actual_qty INT, setup_min INT, cycle_time_min NUMBER(6,2), schedule_adherence NUMBER(5,3), priority INT)
- **Dims**: `dim_date`, `dim_machine`(machine_id, line, type, max_capacity, shift_pattern), `dim_product`(product_id, name, family, bom_complexity, cycle_standard)
- **Grain**: One row per work order per day
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: planned_qty=NORMAL(500,200), setup_min=NORMAL(45,25), cycle_time_min=NORMAL(2.5,0.8), machine_id=UNIFORM, schedule_adherence=NORMAL(0.88,0.1)
- **Dim Sizes**: dim_machine=60, dim_product=100
- **Seasonality**: demand_peak(Sep-Nov, +1.3x), new_product_intro(Q1, +1.2x setup time), shutdown_restart(Jan, −0.3x first week)

## Manufacturing > Energy Management & Sustainability
- **Fact**: `fact_energy_consumption` (consumption_id INT, machine_id INT FK→dim_machine, date_id INT FK→dim_date, kwh_consumed NUMBER(10,2), natural_gas_therms NUMBER(8,2), carbon_kg NUMBER(8,2), production_units INT, energy_per_unit NUMBER(8,4))
- **Dims**: `dim_date`, `dim_machine`(machine_id, line, type, rated_power_kw, vintage_year), `dim_energy_rate`(rate_id, tariff_code, tou_period, rate_per_kwh)
- **Grain**: One row per machine per shift
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: kwh_consumed=NORMAL(850,400), carbon_kg=NORMAL(350,200), production_units=NORMAL(400,150), machine_id=UNIFORM, energy_per_unit=NORMAL(2.1,0.6)
- **Dim Sizes**: dim_machine=60, dim_energy_rate=12
- **Seasonality**: summer_cooling(Jun-Aug, +1.3x HVAC), winter_heating(Dec-Feb, +1.2x gas), weekend_baseload(Sat-Sun, 0.6x)

## Manufacturing > Digital Twin & Process Optimization
- **Fact**: `fact_process_parameter` (param_id INT, equipment_id INT FK→dim_equipment, recipe_id INT FK→dim_recipe, date_id INT FK→dim_date, temperature NUMBER(6,2), pressure NUMBER(8,2), flow_rate NUMBER(8,3), yield_pct NUMBER(5,3), deviation_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_equipment`(equipment_id, type, line, capacity, sensor_count), `dim_recipe`(recipe_id, product, version, target_temp, target_pressure, target_flow)
- **Grain**: One row per equipment per recipe run per hour
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: temperature=NORMAL(180,12), pressure=NORMAL(45,8), flow_rate=NORMAL(12.5,2.0), yield_pct=NORMAL(0.94,0.03), equipment_id=UNIFORM, deviation_flag=UNIFORM(0.04)
- **Dim Sizes**: dim_equipment=80, dim_recipe=40
- **Seasonality**: ambient_impact(Jun-Aug, +0.5°C shift), post_shutdown_drift(Jan, +1.5x deviations), changeover_period(quarterly, +1.2x)

## Manufacturing > Traceability & Genealogy
- **Fact**: `fact_lot_step` (step_id INT, lot_id INT FK→dim_lot, process_step_id INT FK→dim_process_step, date_id INT FK→dim_date, operator_id INT, equipment_id INT, qty_in INT, qty_out INT, scrap_qty INT, hold_flag BOOLEAN, quality_result VARCHAR)
- **Dims**: `dim_date`, `dim_lot`(lot_id, material_id, parent_lot_id, batch_size, supplier_id, expiry_date), `dim_process_step`(step_id, name, sequence, department, spc_enabled)
- **Grain**: One row per lot per process step
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: qty_in=NORMAL(1000,300), scrap_qty=ZIPF(2.5), lot_id=UNIFORM, process_step_id=UNIFORM, hold_flag=UNIFORM(0.03)
- **Dim Sizes**: dim_lot=2000, dim_process_step=25
- **Seasonality**: new_supplier_lots(Q1, +1.2x holds), audit_season(varies, +1.3x documentation), recall_events(random, +5.0x trace queries)
- **Correlated FK**: lot_id → dim_lot → parent_lot_id (derive parent genealogy via recursive lookup)

## Manufacturing > Workforce Productivity & Safety
- **Fact**: `fact_shift_output` (shift_output_id INT, operator_id INT FK→dim_operator, machine_id INT FK→dim_machine, date_id INT FK→dim_date, units_produced INT, defects_produced INT, safety_incidents INT, overtime_hours NUMBER(4,1), productivity_pct NUMBER(5,3))
- **Dims**: `dim_date`, `dim_operator`(operator_id, name, shift, certification_level, department, hire_date), `dim_machine`(machine_id, line, type, complexity_tier)
- **Grain**: One row per operator per shift
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: units_produced=NORMAL(450,100), defects_produced=ZIPF(2.5), safety_incidents=ZIPF(3.0), operator_id=UNIFORM, productivity_pct=NORMAL(0.88,0.1), overtime_hours=NORMAL(1.5,1.2)
- **Dim Sizes**: dim_operator=150, dim_machine=60
- **Seasonality**: holiday_overtime(Nov-Dec, +1.5x OT), summer_temp_workers(Jun-Aug, +1.2x workforce, −0.1x productivity), monday_incidents(Mon, +1.3x safety)

## Manufacturing > Connected Product & After-Sales Analytics
- **Fact**: `fact_product_telemetry` (telemetry_id INT, product_instance_id INT FK→dim_product_instance, date_id INT FK→dim_date, operating_hours NUMBER(6,1), error_count INT, performance_score NUMBER(5,2), service_due_flag BOOLEAN, warranty_claim_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_product_instance`(instance_id, model, serial_number, customer_id, install_date, warranty_end_date, region), `dim_failure_mode`(mode_id, name, category, severity, mtbf_hours)
- **Grain**: One row per product instance per day
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: operating_hours=NORMAL(8,4), error_count=ZIPF(2.5), performance_score=NORMAL(85,12), product_instance_id=UNIFORM, service_due_flag=UNIFORM(0.03), warranty_claim_flag=UNIFORM(0.008)
- **Dim Sizes**: dim_product_instance=1000, dim_failure_mode=30
- **Seasonality**: summer_heat_failures(Jun-Aug, +1.3x errors), post_warranty_cliff(month 13, +1.5x claims), winter_usage_dip(Dec, 0.85x hours)

## Telecom > Next-Best-Action & Personalized Offers
- **Fact**: `fact_offer_event` (offer_id INT, subscriber_id INT FK→dim_subscriber, campaign_id INT FK→dim_campaign, date_id INT FK→dim_date, channel VARCHAR, propensity_score NUMBER(5,3), presented BOOLEAN, accepted BOOLEAN, uplift_arpu NUMBER(6,2))
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, plan_type, tenure_months, arpu_band, segment), `dim_campaign`(campaign_id, name, offer_type, target_segment, discount_pct)
- **Grain**: One row per offer presented to subscriber
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: propensity_score=NORMAL(0.2,0.15), uplift_arpu=NORMAL(8,5), subscriber_id=ZIPF(1.1), campaign_id=ZIPF(1.1), accepted=UNIFORM(0.12)
- **Dim Sizes**: dim_subscriber=500, dim_campaign=50
- **Seasonality**: contract_renewal_window(month 11-12, +2.0x), holiday_promo(Dec, +1.5x), back_to_school(Aug-Sep, +1.2x)

## Telecom > IoT & Connected Device Analytics
- **Fact**: `fact_iot_session` (session_id INT, device_id INT FK→dim_device, date_id INT FK→dim_date, data_bytes NUMBER(12,0), session_duration_sec INT, signal_strength NUMBER(5,1), connection_drops INT, protocol VARCHAR)
- **Dims**: `dim_date`, `dim_device`(device_id, type, manufacturer, sim_id, customer_id, activation_date, use_case), `dim_service_plan`(plan_id, name, data_cap_mb, sla_uptime_pct)
- **Grain**: One row per device per session (aggregated daily)
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: data_bytes=ZIPF(1.7), session_duration_sec=NORMAL(300,200), signal_strength=NORMAL(-75,12), device_id=UNIFORM, connection_drops=ZIPF(2.5)
- **Dim Sizes**: dim_device=2000, dim_service_plan=10
- **Seasonality**: winter_connectivity_issues(Dec-Feb, +1.3x drops), summer_peak_usage(Jun-Aug, +1.2x data), new_year_activations(Jan, +1.4x devices)

## Telecom > Customer Experience & NPS Analytics
- **Fact**: `fact_cx_signal` (signal_id INT, subscriber_id INT FK→dim_subscriber, touchpoint_id INT FK→dim_touchpoint, date_id INT FK→dim_date, nps_score INT, csat_score NUMBER(3,1), effort_score INT, sentiment VARCHAR, resolution_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_subscriber`(subscriber_id, tenure_months, plan_type, complaint_history_count), `dim_touchpoint`(touchpoint_id, channel, category, department, avg_handle_min)
- **Grain**: One row per customer experience measurement
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: nps_score=NORMAL(32,25), csat_score=NORMAL(3.8,0.9), effort_score=NORMAL(3.2,1.2), subscriber_id=ZIPF(1.1), resolution_flag=UNIFORM(0.72)
- **Dim Sizes**: dim_subscriber=500, dim_touchpoint=30
- **Seasonality**: post_price_increase(event, +2.0x negative), network_outage_aftermath(event, +3.0x), holiday_travel_complaints(Dec, +1.3x)

## Telecom > 5G Monetization & Network Slicing
- **Fact**: `fact_slice_usage` (usage_id INT, slice_id INT FK→dim_slice, enterprise_id INT FK→dim_enterprise, date_id INT FK→dim_date, throughput_gbps NUMBER(8,3), latency_ms NUMBER(6,2), uptime_pct NUMBER(5,4), billed_amount NUMBER(10,2), sla_breach_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_slice`(slice_id, name, type, sla_latency_ms, sla_throughput_gbps, reserved_capacity_pct), `dim_enterprise`(enterprise_id, name, industry, contract_value, tier)
- **Grain**: One row per slice per enterprise per hour
- **Volume**: ~350K rows/year at medium scale
- **Distributions**: throughput_gbps=NORMAL(2.5,1.5), latency_ms=NORMAL(5,3), uptime_pct=NORMAL(0.9995,0.001), billed_amount=ZIPF(1.8), enterprise_id=ZIPF(1.1), sla_breach_flag=UNIFORM(0.002)
- **Dim Sizes**: dim_slice=20, dim_enterprise=50
- **Seasonality**: event_surge(concerts/sports, +3.0x capacity), business_hours(9-17h, +1.5x enterprise), year_end_contracts(Dec, +1.4x new slices)
- **Intraday**: Pattern #13 business + event overlay (peaks 9-17h, event spikes 18-22h)

## Telecom > Partner & Wholesale Settlement
- **Fact**: `fact_settlement_record` (record_id INT, partner_id INT FK→dim_partner, date_id INT FK→dim_date, cdr_count INT, rated_amount NUMBER(10,4), billed_amount NUMBER(10,4), variance NUMBER(8,4), dispute_flag BOOLEAN, settlement_type VARCHAR)
- **Dims**: `dim_date`, `dim_partner`(partner_id, name, type, country, interconnect_agreement_id, rate_card_version), `dim_service_type`(service_type_id, name, direction, unit)
- **Grain**: One row per partner per service type per day
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: cdr_count=ZIPF(1.6), rated_amount=ZIPF(1.7), variance=NORMAL(0,0.005), partner_id=ZIPF(1.1), dispute_flag=UNIFORM(0.03)
- **Dim Sizes**: dim_partner=40, dim_service_type=15
- **Seasonality**: roaming_peak(Jun-Aug, +1.6x international), month_end_reconciliation(last 2 days, +2.0x), rate_card_change(Jan+Jul, +1.5x disputes)

## Telecom > Predictive Maintenance (Network Infrastructure)
- **Fact**: `fact_equipment_health` (health_id INT, equipment_id INT FK→dim_equipment, date_id INT FK→dim_date, alarm_count INT, error_rate NUMBER(5,4), degradation_score NUMBER(5,2), temperature_c NUMBER(5,1), mtbf_remaining_days INT, maintenance_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_equipment`(equipment_id, type, vendor, site_id, install_date, firmware_version, criticality), `dim_alarm_type`(alarm_id, name, severity, category)
- **Grain**: One row per equipment per day
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: alarm_count=ZIPF(2.5), error_rate=NORMAL(0.001,0.002), degradation_score=NORMAL(25,18), equipment_id=UNIFORM, maintenance_flag=UNIFORM(0.015)
- **Dim Sizes**: dim_equipment=800, dim_alarm_type=40
- **Seasonality**: summer_heat_stress(Jun-Aug, +1.4x failures), storm_damage(varies, +2.0x alarms), firmware_rollout(quarterly, +1.3x errors)

## Energy & Utilities > Renewable Integration & Curtailment
- **Fact**: `fact_generation_interval` (interval_id INT, plant_id INT FK→dim_plant, date_id INT FK→dim_date, hour INT, generation_mwh NUMBER(10,3), curtailed_mwh NUMBER(10,3), capacity_factor NUMBER(5,3), price_per_mwh NUMBER(8,2), grid_congestion_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_plant`(plant_id, name, type, capacity_mw, location, interconnection_year), `dim_weather`(weather_id, irradiance_wm2, wind_speed_ms, cloud_cover_pct)
- **Grain**: One row per generation plant per hour
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: generation_mwh=NORMAL(35,20), curtailed_mwh=ZIPF(2.5), capacity_factor=NORMAL(0.28,0.12), plant_id=UNIFORM, grid_congestion_flag=UNIFORM(0.08)
- **Dim Sizes**: dim_plant=50, dim_weather=365
- **Seasonality**: summer_solar_peak(Jun-Aug, +1.5x solar), winter_wind(Nov-Feb, +1.4x wind), spring_curtailment(Mar-May, +2.0x curtailment—low demand + high gen)

## Energy & Utilities > Asset Health & Predictive Maintenance
- **Fact**: `fact_asset_condition` (condition_id INT, asset_id INT FK→dim_asset, date_id INT FK→dim_date, health_index NUMBER(5,2), dga_hydrogen_ppm NUMBER(6,1), load_pct NUMBER(5,3), age_years INT, failure_probability NUMBER(5,4), inspection_due_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_asset`(asset_id, type, substation, voltage_kv, manufacturer, install_year, criticality), `dim_inspection`(inspection_id, type, method, interval_months)
- **Grain**: One row per asset per month
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: health_index=NORMAL(72,18), dga_hydrogen_ppm=ZIPF(2.0), load_pct=NORMAL(0.6,0.2), asset_id=UNIFORM, failure_probability=NORMAL(0.01,0.02), inspection_due_flag=UNIFORM(0.08)
- **Dim Sizes**: dim_asset=500, dim_inspection=20
- **Seasonality**: summer_loading(Jun-Aug, +1.3x load), storm_damage(varies, +2.0x failures), spring_inspection(Mar-May, +1.5x inspections)

## Energy & Utilities > Customer Segmentation & Rate Optimization
- **Fact**: `fact_load_profile` (profile_id INT, customer_id INT FK→dim_customer, rate_id INT FK→dim_rate, date_id INT FK→dim_date, peak_kwh NUMBER(8,2), offpeak_kwh NUMBER(8,2), demand_kw NUMBER(8,2), bill_amount NUMBER(8,2), solar_export_kwh NUMBER(8,2))
- **Dims**: `dim_date`, `dim_customer`(customer_id, class, segment, zip, solar_flag, ev_flag, heating_type), `dim_rate`(rate_id, tariff_code, name, tou_flag, demand_charge_rate)
- **Grain**: One row per customer per billing month
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: peak_kwh=NORMAL(450,250), offpeak_kwh=NORMAL(650,300), demand_kw=ZIPF(1.7), bill_amount=NORMAL(120,65), customer_id=UNIFORM, solar_export_kwh=ZIPF(2.0)
- **Dim Sizes**: dim_customer=1000, dim_rate=15
- **Seasonality**: summer_peak(Jun-Aug, +1.5x peak_kwh), winter_heating(Dec-Feb, +1.3x), spring_low(Apr-May, 0.75x)

## Energy & Utilities > Distributed Energy Resource (DER) Management
- **Fact**: `fact_der_reading` (reading_id INT, der_id INT FK→dim_der, date_id INT FK→dim_date, hour INT, generation_kwh NUMBER(8,3), export_kwh NUMBER(8,3), state_of_charge_pct NUMBER(5,2), grid_injection_flag BOOLEAN, dispatch_signal VARCHAR)
- **Dims**: `dim_date`, `dim_der`(der_id, type, capacity_kw, customer_id, feeder_id, install_date, inverter_model), `dim_program`(program_id, name, type, incentive_rate)
- **Grain**: One row per DER per hour
- **Volume**: ~500K rows/year at medium scale
- **Distributions**: generation_kwh=NORMAL(3.5,2.0), export_kwh=NORMAL(1.5,1.2), state_of_charge_pct=NORMAL(0.55,0.25), der_id=UNIFORM, grid_injection_flag=UNIFORM(0.35)
- **Dim Sizes**: dim_der=500, dim_program=10
- **Seasonality**: summer_solar(Jun-Aug, +1.6x generation), winter_battery_draw(Dec-Feb, +1.3x discharge), cloud_variability(spring, +1.5x ramp events)
- **Intraday**: Pattern #13 solar bell curve (peak 10-14h generation, evening battery discharge 17-21h)

## Energy & Utilities > Regulatory Compliance & Rate Case
- **Fact**: `fact_cost_allocation` (allocation_id INT, cost_center_id INT FK→dim_cost_center, rate_class_id INT FK→dim_rate_class, date_id INT FK→dim_date, revenue_requirement NUMBER(12,2), allocated_cost NUMBER(12,2), rate_base NUMBER(14,2), depreciation NUMBER(10,2), return_on_equity_pct NUMBER(5,3))
- **Dims**: `dim_date`, `dim_cost_center`(cc_id, name, ferc_account, category, department), `dim_rate_class`(class_id, name, customer_count, avg_usage_kwh, demand_profile)
- **Grain**: One row per cost center per rate class per month
- **Volume**: ~50K rows/year at medium scale
- **Distributions**: revenue_requirement=ZIPF(1.7), allocated_cost=ZIPF(1.8), rate_base=ZIPF(1.6), cost_center_id=UNIFORM, return_on_equity_pct=NORMAL(0.10,0.01)
- **Dim Sizes**: dim_cost_center=40, dim_rate_class=10
- **Seasonality**: rate_case_filing(varies, +2.0x activity), annual_true_up(Dec, +1.5x), test_year_prep(quarterly, +1.3x)

## Energy & Utilities > Vegetation Management & Wildfire Risk
- **Fact**: `fact_veg_risk_score` (score_id INT, span_id INT FK→dim_span, date_id INT FK→dim_date, risk_score NUMBER(5,2), tree_proximity_ft NUMBER(6,1), growth_rate_in_yr NUMBER(4,1), fire_weather_index NUMBER(5,1), trim_priority INT, outage_history_count INT)
- **Dims**: `dim_date`, `dim_span`(span_id, feeder_id, circuit, length_mi, voltage_kv, fire_zone, tree_density, last_trim_date), `dim_fire_zone`(zone_id, name, risk_tier, cpuc_tier, wind_corridor_flag)
- **Grain**: One row per span per month
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: risk_score=NORMAL(40,22), tree_proximity_ft=NORMAL(8,5), fire_weather_index=NORMAL(35,20), span_id=UNIFORM, trim_priority=ZIPF(2.0), outage_history_count=ZIPF(2.5)
- **Dim Sizes**: dim_span=1000, dim_fire_zone=25
- **Seasonality**: fire_season(Jun-Oct, +2.0x risk), growth_season(Mar-Jun, +1.5x growth), dormant_winter(Dec-Feb, 0.7x growth), wind_events(Oct-Nov, +2.5x fire weather)

## Energy & Utilities > Electric Vehicle Load Impact
- **Fact**: `fact_ev_charging` (charging_id INT, charger_id INT FK→dim_charger, date_id INT FK→dim_date, hour INT, energy_kwh NUMBER(8,2), session_duration_min INT, transformer_load_pct NUMBER(5,3), managed_flag BOOLEAN, rate_type VARCHAR)
- **Dims**: `dim_date`, `dim_charger`(charger_id, type, location, feeder_id, transformer_id, rated_kw), `dim_transformer`(transformer_id, nameplate_kva, current_peak_kw, headroom_pct, upgrade_year)
- **Grain**: One row per charger per hour
- **Volume**: ~350K rows/year at medium scale
- **Distributions**: energy_kwh=NORMAL(12,7), session_duration_min=NORMAL(180,90), transformer_load_pct=NORMAL(0.6,0.2), charger_id=UNIFORM, managed_flag=UNIFORM(0.3)
- **Dim Sizes**: dim_charger=200, dim_transformer=100
- **Seasonality**: winter_range_anxiety(Dec-Feb, +1.2x charging), summer_road_trips(Jun-Aug, +1.1x), evening_home_peak(17-22h, +2.0x residential)
- **Intraday**: Pattern #13 EV charging (residential peak 18-22h, workplace peak 8-10h)

## Logistics & Transportation > Demand-Capacity Matching
- **Fact**: `fact_lane_demand` (demand_id INT, lane_id INT FK→dim_lane, date_id INT FK→dim_date, shipment_count INT, total_weight_lbs NUMBER(10,0), available_capacity INT, utilization_pct NUMBER(5,3), spot_rate NUMBER(8,2), contract_rate NUMBER(8,2))
- **Dims**: `dim_date`, `dim_lane`(lane_id, origin_city, dest_city, distance_mi, mode, avg_transit_days), `dim_capacity`(capacity_id, vehicle_type, max_weight, max_cube)
- **Grain**: One row per lane per day
- **Volume**: ~250K rows/year at medium scale
- **Distributions**: shipment_count=NORMAL(45,25), total_weight_lbs=NORMAL(35000,15000), utilization_pct=NORMAL(0.78,0.15), lane_id=ZIPF(1.1), spot_rate=NORMAL(2.5,0.8)
- **Dim Sizes**: dim_lane=300, dim_capacity=15
- **Seasonality**: peak_shipping(Oct-Dec, +1.6x), produce_season(Apr-Jun, +1.3x reefer), january_lull(Jan, 0.7x), deadhead_imbalance(varies by lane)

## Logistics & Transportation > Carrier Performance & Procurement
- **Fact**: `fact_carrier_scorecard` (scorecard_id INT, carrier_id INT FK→dim_carrier, lane_id INT FK→dim_lane, date_id INT FK→dim_date, shipments_tendered INT, shipments_accepted INT, on_time_pct NUMBER(5,3), damage_rate NUMBER(5,4), avg_cost_per_mile NUMBER(6,4), claims_count INT)
- **Dims**: `dim_date`, `dim_carrier`(carrier_id, name, type, mc_number, fleet_size, insurance_rating, safety_score), `dim_lane`(lane_id, origin, destination, distance_mi, mode)
- **Grain**: One row per carrier per lane per week
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: shipments_tendered=NORMAL(20,12), on_time_pct=NORMAL(0.88,0.08), damage_rate=NORMAL(0.005,0.005), carrier_id=ZIPF(1.1), avg_cost_per_mile=NORMAL(2.8,0.6), claims_count=ZIPF(2.5)
- **Dim Sizes**: dim_carrier=100, dim_lane=200
- **Seasonality**: tight_market(Oct-Dec, +1.4x rejection), bid_season(Jan-Mar, +2.0x RFP activity), summer_capacity(Jun-Aug, 0.9x cost)

## Logistics & Transportation > Warehouse Operations Analytics
- **Fact**: `fact_warehouse_activity` (activity_id INT, warehouse_id INT FK→dim_warehouse, date_id INT FK→dim_date, inbound_pallets INT, outbound_pallets INT, picks_completed INT, picks_per_hour NUMBER(6,1), dock_dwell_min NUMBER(6,1), labor_hours NUMBER(6,1))
- **Dims**: `dim_date`, `dim_warehouse`(warehouse_id, name, region, sqft, dock_doors, automation_level), `dim_zone`(zone_id, name, type, pick_method, capacity_pallets)
- **Grain**: One row per warehouse per day
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: inbound_pallets=NORMAL(350,150), outbound_pallets=NORMAL(400,180), picks_per_hour=NORMAL(120,35), warehouse_id=UNIFORM, dock_dwell_min=NORMAL(45,25), labor_hours=NORMAL(200,80)
- **Dim Sizes**: dim_warehouse=20, dim_zone=50
- **Seasonality**: peak_season(Oct-Dec, +1.7x), post_holiday_returns(Jan, +1.3x inbound), summer_steady(Jun-Aug, 0.9x)

## Logistics & Transportation > Last-Mile Delivery Optimization
- **Fact**: `fact_delivery_attempt` (attempt_id INT, route_id INT FK→dim_route, date_id INT FK→dim_date, stops_planned INT, stops_completed INT, failed_deliveries INT, total_miles NUMBER(8,1), cost_per_stop NUMBER(6,2), customer_rating NUMBER(3,1), eta_accuracy_min NUMBER(6,1))
- **Dims**: `dim_date`, `dim_route`(route_id, zone, density_type, vehicle_type, max_stops), `dim_delivery_zone`(zone_id, name, urban_rural, avg_density_stops_per_mile, access_difficulty)
- **Grain**: One row per route per day
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: stops_planned=NORMAL(85,30), failed_deliveries=ZIPF(2.5), cost_per_stop=NORMAL(8.5,3.0), route_id=UNIFORM, eta_accuracy_min=NORMAL(12,8), customer_rating=NORMAL(4.2,0.6)
- **Dim Sizes**: dim_route=200, dim_delivery_zone=40
- **Seasonality**: holiday_peak(Nov-Dec, +2.0x volume), amazon_prime_day(event, +3.0x), weather_disruption(winter, +1.5x failed), summer_normal(Jun-Aug, 0.9x)

## Logistics & Transportation > Carbon Footprint & Sustainability
- **Fact**: `fact_shipment_emission` (emission_id INT, shipment_id INT FK→dim_shipment, date_id INT FK→dim_date, distance_km NUMBER(8,1), fuel_liters NUMBER(8,2), co2_kg NUMBER(8,2), mode VARCHAR, load_factor_pct NUMBER(5,3), offset_purchased BOOLEAN)
- **Dims**: `dim_date`, `dim_shipment`(shipment_id, origin, destination, customer_id, weight_kg, mode, vehicle_class), `dim_emission_factor`(factor_id, mode, vehicle_class, fuel_type, co2_per_km)
- **Grain**: One row per shipment leg
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: distance_km=NORMAL(500,400), co2_kg=NORMAL(150,120), fuel_liters=NORMAL(60,45), shipment_id=UNIFORM, load_factor_pct=NORMAL(0.72,0.15), offset_purchased=UNIFORM(0.05)
- **Dim Sizes**: dim_shipment=5000, dim_emission_factor=30
- **Seasonality**: peak_emissions(Oct-Dec, +1.5x volume), summer_efficiency(Jun-Aug, −0.1x per-unit from lighter loads), reporting_deadline(Jan-Mar, +1.3x data collection)

## Logistics & Transportation > Claims & Damage Analytics
- **Fact**: `fact_freight_claim` (claim_id INT, shipment_id INT FK→dim_shipment, carrier_id INT FK→dim_carrier, date_id INT FK→dim_date, claim_amount NUMBER(10,2), recovered_amount NUMBER(10,2), damage_type VARCHAR, days_to_resolve INT, liable_party VARCHAR)
- **Dims**: `dim_date`, `dim_shipment`(shipment_id, origin, destination, commodity_type, weight_lbs, handling_class), `dim_carrier`(carrier_id, name, type, claims_history_count, insurance_limit)
- **Grain**: One row per freight claim
- **Volume**: ~25K rows/year at medium scale
- **Distributions**: claim_amount=ZIPF(1.8), recovered_amount=ZIPF(2.0), days_to_resolve=NORMAL(45,30), shipment_id=UNIFORM, carrier_id=ZIPF(1.1)
- **Dim Sizes**: dim_shipment=3000, dim_carrier=80
- **Seasonality**: peak_damage(Oct-Dec, +1.6x volume/rush handling), winter_weather(Dec-Feb, +1.3x), summer_heat(Jun-Aug, +1.2x perishable)

## Logistics & Transportation > Network Design & Lane Optimization
- **Fact**: `fact_network_flow` (flow_id INT, origin_id INT FK→dim_facility, dest_id INT FK→dim_facility, date_id INT FK→dim_date, volume_units INT, transit_days NUMBER(4,1), cost_total NUMBER(10,2), mode VARCHAR, consolidation_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_facility`(facility_id, name, type, city, state, lat, lon, capacity, fixed_cost_monthly)
- **Grain**: One row per origin-destination pair per week
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: volume_units=ZIPF(1.7), transit_days=NORMAL(3.5,1.5), cost_total=ZIPF(1.8), origin_id=ZIPF(1.1), dest_id=ZIPF(1.1), consolidation_flag=UNIFORM(0.25)
- **Dim Sizes**: dim_facility=50
- **Seasonality**: peak_volume(Oct-Dec, +1.5x), post_peak_repositioning(Jan, +1.3x empty moves), quarterly_review(Mar/Jun/Sep/Dec, +1.2x analysis)
- **Note**: origin_id and dest_id both reference dim_facility (self-referencing dimension for network modeling)

## Media & Entertainment > Content Performance & ROI
- **Fact**: `fact_content_metric` (metric_id INT, content_id INT FK→dim_content, date_id INT FK→dim_date, view_hours NUMBER(10,1), unique_viewers INT, completion_rate NUMBER(5,3), acquisition_attributed INT, cost_per_hour NUMBER(8,2), revenue_attributed NUMBER(10,2))
- **Dims**: `dim_date`, `dim_content`(content_id, title, type, genre, release_date, production_cost, license_cost, original_flag), `dim_market`(market_id, name, region, subscriber_base)
- **Grain**: One row per content per market per week
- **Volume**: ~300K rows/year at medium scale
- **Distributions**: view_hours=ZIPF(1.8), unique_viewers=ZIPF(1.7), completion_rate=NORMAL(0.55,0.2), content_id=ZIPF(1.1), cost_per_hour=ZIPF(1.9)
- **Dim Sizes**: dim_content=500, dim_market=20
- **Seasonality**: release_window_spike(first 2 weeks, +3.0x), holiday_binge(Dec, +1.5x), summer_catalog(Jun-Aug, +1.2x library content)

## Media & Entertainment > Rights & Royalty Management
- **Fact**: `fact_royalty_accrual` (accrual_id INT, content_id INT FK→dim_content, licensor_id INT FK→dim_licensor, date_id INT FK→dim_date, streams_count INT, royalty_amount NUMBER(10,4), territory VARCHAR, window_type VARCHAR, expired_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_content`(content_id, title, type, rights_start_date, rights_end_date), `dim_licensor`(licensor_id, name, type, payment_terms_days, min_guarantee)
- **Grain**: One row per content per licensor per territory per month
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: streams_count=ZIPF(1.8), royalty_amount=ZIPF(1.9), content_id=ZIPF(1.1), licensor_id=ZIPF(1.1), expired_flag=UNIFORM(0.03)
- **Dim Sizes**: dim_content=500, dim_licensor=100
- **Seasonality**: quarter_end_settlement(Mar/Jun/Sep/Dec, +2.0x payments), rights_renewal(varies, +1.5x), new_content_window(monthly, +1.3x)

## Media & Entertainment > Content Demand Forecasting
- **Fact**: `fact_demand_signal` (signal_id INT, content_id INT FK→dim_content, date_id INT FK→dim_date, social_mentions INT, search_volume INT, trailer_views INT, pre_save_count INT, demand_index NUMBER(5,2), predicted_viewers INT)
- **Dims**: `dim_date`, `dim_content`(content_id, title, genre, talent_score, franchise_flag, release_date, budget_tier), `dim_platform`(platform_id, name, type, reach)
- **Grain**: One row per content per platform per day
- **Volume**: ~400K rows/year at medium scale
- **Distributions**: social_mentions=ZIPF(1.9), search_volume=ZIPF(1.8), demand_index=NORMAL(35,22), content_id=ZIPF(1.1), predicted_viewers=ZIPF(1.7)
- **Dim Sizes**: dim_content=300, dim_platform=10
- **Seasonality**: pre_release_ramp(−30d to release, +3.0x), awards_season(Jan-Mar, +1.5x nominated), summer_blockbuster(Jun-Aug, +1.3x)

## Media & Entertainment > Real-Time Personalization
- **Fact**: `fact_personalization_event` (event_id INT, user_id INT FK→dim_user, date_id INT FK→dim_date, surface VARCHAR, recommendations_shown INT, position_clicked INT, dwell_sec INT, conversion_flag BOOLEAN, algorithm_version VARCHAR)
- **Dims**: `dim_date`, `dim_user`(user_id, plan_tier, cohort_month, device_primary, preference_vector), `dim_experiment`(experiment_id, name, variant, start_date, end_date)
- **Grain**: One row per personalization impression
- **Volume**: ~1000K rows/year at medium scale
- **Distributions**: recommendations_shown=NORMAL(12,4), position_clicked=ZIPF(2.0), dwell_sec=NORMAL(15,10), user_id=ZIPF(1.1), conversion_flag=UNIFORM(0.08)
- **Dim Sizes**: dim_user=1000, dim_experiment=30
- **Seasonality**: evening_engagement(19-23h, +2.0x), weekend_browsing(Sat-Sun, +1.4x), new_content_day(release day, +1.8x)
- **Intraday**: Pattern #13 evening peak (ramp 17-19h, peak 20-22h, decline 23-01h)

## Media & Entertainment > Creator/Talent Analytics
- **Fact**: `fact_creator_performance` (perf_id INT, creator_id INT FK→dim_creator, date_id INT FK→dim_date, content_published INT, total_views INT, engagement_rate NUMBER(5,3), revenue_generated NUMBER(10,2), audience_growth_pct NUMBER(5,3), brand_deal_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_creator`(creator_id, name, category, tier, audience_size, contract_type, exclusive_flag), `dim_genre`(genre_id, name, avg_engagement, growth_trend)
- **Grain**: One row per creator per week
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: total_views=ZIPF(1.9), engagement_rate=NORMAL(0.04,0.03), revenue_generated=ZIPF(2.0), creator_id=ZIPF(1.1), audience_growth_pct=NORMAL(0.02,0.05), brand_deal_flag=UNIFORM(0.05)
- **Dim Sizes**: dim_creator=200, dim_genre=20
- **Seasonality**: q4_brand_deals(Oct-Dec, +1.8x), summer_content_dip(Jul, 0.85x), viral_spikes(random, +5.0x single creator)

## Media & Entertainment > Audience Segmentation & Monetization
- **Fact**: `fact_audience_value` (value_id INT, segment_id INT FK→dim_segment, date_id INT FK→dim_date, segment_size INT, avg_watch_hours NUMBER(6,1), ad_revenue_per_user NUMBER(8,4), subscription_revenue_per_user NUMBER(8,2), churn_rate NUMBER(5,3), cpm_achieved NUMBER(8,2))
- **Dims**: `dim_date`, `dim_segment`(segment_id, name, demographic_profile, behavioral_cluster, value_tier, addressable_flag), `dim_ad_product`(product_id, name, format, targeting_type, min_cpm)
- **Grain**: One row per audience segment per ad product per week
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: segment_size=ZIPF(1.6), avg_watch_hours=NORMAL(12,6), ad_revenue_per_user=NORMAL(2.5,1.5), cpm_achieved=NORMAL(14,8), segment_id=UNIFORM
- **Dim Sizes**: dim_segment=50, dim_ad_product=20
- **Seasonality**: q4_ad_premium(Oct-Dec, +1.8x CPM), january_budget_reset(Jan, 0.5x), upfront_season(May-Jun, +1.3x commitments)

## Life Sciences & Pharma > Drug Supply Chain & Cold Chain
- **Fact**: `fact_shipment_condition` (condition_id INT, lot_id INT FK→dim_lot, warehouse_id INT FK→dim_warehouse, date_id INT FK→dim_date, temperature_avg_c NUMBER(5,2), temperature_max_c NUMBER(5,2), excursion_flag BOOLEAN, transit_hours INT, qty_units INT, disposition VARCHAR)
- **Dims**: `dim_date`, `dim_lot`(lot_id, drug_id, batch_number, manufacture_date, expiry_date, serial_range), `dim_warehouse`(warehouse_id, name, type, region, temperature_zone, gdp_certified)
- **Grain**: One row per lot per warehouse per day
- **Volume**: ~200K rows/year at medium scale
- **Distributions**: temperature_avg_c=NORMAL(4,2), transit_hours=NORMAL(36,24), qty_units=NORMAL(5000,3000), lot_id=UNIFORM, excursion_flag=UNIFORM(0.02)
- **Dim Sizes**: dim_lot=1000, dim_warehouse=30
- **Seasonality**: summer_excursions(Jun-Aug, +2.0x temperature events), flu_vaccine_peak(Aug-Oct, +1.5x volume), year_end_inventory(Dec, +1.3x)

## Life Sciences & Pharma > Manufacturing Batch Quality & Deviation
- **Fact**: `fact_batch_result` (result_id INT, batch_id INT FK→dim_batch, test_id INT FK→dim_quality_test, date_id INT FK→dim_date, result_value NUMBER(10,4), spec_min NUMBER(10,4), spec_max NUMBER(10,4), pass_flag BOOLEAN, deviation_flag BOOLEAN, capa_id INT)
- **Dims**: `dim_date`, `dim_batch`(batch_id, product_id, lot_number, batch_size, equipment_train, start_date), `dim_quality_test`(test_id, name, method, category, critical_flag)
- **Grain**: One row per batch per quality test
- **Volume**: ~150K rows/year at medium scale
- **Distributions**: result_value=NORMAL(target,0.05*target), batch_id=UNIFORM, test_id=UNIFORM, pass_flag=UNIFORM(0.96), deviation_flag=UNIFORM(0.04)
- **Dim Sizes**: dim_batch=500, dim_quality_test=60
- **Seasonality**: post_shutdown_startup(Jan, +1.5x deviations), new_material_lot(Q1, +1.2x OOS), campaign_changeover(varies, +1.3x)
- **Note**: result_value distribution depends on the specific test—use target from dim_quality_test and noise proportional to spec range

## Life Sciences & Pharma > Regulatory Submission Analytics
- **Fact**: `fact_submission_milestone` (milestone_id INT, submission_id INT FK→dim_submission, document_id INT FK→dim_document, date_id INT FK→dim_date, status VARCHAR, days_elapsed INT, reviewer_queries INT, gap_flag BOOLEAN, target_date DATE)
- **Dims**: `dim_date`, `dim_submission`(submission_id, type, agency, therapeutic_area, phase, target_action_date), `dim_document`(document_id, module, section, version, page_count)
- **Grain**: One row per submission per document milestone
- **Volume**: ~30K rows/year at medium scale
- **Distributions**: days_elapsed=NORMAL(90,45), reviewer_queries=ZIPF(2.5), submission_id=ZIPF(1.1), gap_flag=UNIFORM(0.08)
- **Dim Sizes**: dim_submission=50, dim_document=200
- **Seasonality**: pdufa_deadline_push(varies, +2.0x activity), year_end_filing(Dec, +1.5x), q1_correspondence(Jan-Mar, +1.3x agency queries)

## Life Sciences & Pharma > Patient Recruitment & Enrollment Prediction
- **Fact**: `fact_screening_event` (screening_id INT, site_id INT FK→dim_site, trial_id INT FK→dim_trial, date_id INT FK→dim_date, screened INT, eligible INT, consented INT, randomized INT, screen_fail_reason VARCHAR, days_to_consent INT)
- **Dims**: `dim_date`, `dim_site`(site_id, name, country, pi_name, capacity, activation_date, referral_network_size), `dim_trial`(trial_id, protocol, phase, therapeutic_area, target_n, diversity_target_pct)
- **Grain**: One row per site per trial per week
- **Volume**: ~60K rows/year at medium scale
- **Distributions**: screened=NORMAL(10,6), eligible=NORMAL(5,3), consented=NORMAL(3,2), site_id=ZIPF(1.1), days_to_consent=NORMAL(7,5), randomized=NORMAL(2,1.5)
- **Dim Sizes**: dim_site=100, dim_trial=20
- **Seasonality**: summer_slowdown(Jul-Aug, −0.3x EU sites), holiday_pause(Dec, −0.4x), new_site_activations(Q1, +1.4x), competing_trial_impact(varies, −0.2x)

## Life Sciences & Pharma > Medical Affairs & Scientific Insights
- **Fact**: `fact_medical_interaction` (interaction_id INT, kol_id INT FK→dim_kol, date_id INT FK→dim_date, interaction_type VARCHAR, topic VARCHAR, publications_cited INT, inquiry_flag BOOLEAN, sentiment VARCHAR, follow_up_required BOOLEAN)
- **Dims**: `dim_date`, `dim_kol`(kol_id, name, specialty, institution, tier, h_index, region), `dim_therapeutic_area`(ta_id, name, pipeline_stage, competitive_intensity)
- **Grain**: One row per medical affairs interaction
- **Volume**: ~40K rows/year at medium scale
- **Distributions**: publications_cited=ZIPF(2.5), kol_id=ZIPF(1.1), inquiry_flag=UNIFORM(0.25), follow_up_required=UNIFORM(0.4)
- **Dim Sizes**: dim_kol=300, dim_therapeutic_area=15
- **Seasonality**: congress_season(ASCO Jun, ASH Dec, +3.0x), publication_cycle(quarterly, +1.3x), launch_period(first 12 months, +2.0x)

## Life Sciences & Pharma > Pricing & Market Access Analytics
- **Fact**: `fact_market_access_event` (event_id INT, drug_id INT FK→dim_drug, payer_id INT FK→dim_payer, date_id INT FK→dim_date, formulary_status VARCHAR, contract_rebate_pct NUMBER(5,3), net_price NUMBER(10,2), lives_covered INT, market_share NUMBER(5,3), access_barrier VARCHAR)
- **Dims**: `dim_date`, `dim_drug`(drug_id, name, therapeutic_class, launch_date, list_price, manufacturer), `dim_payer`(payer_id, name, type, lives, formulary_committee_cycle, region)
- **Grain**: One row per drug per payer per quarter
- **Volume**: ~30K rows/year at medium scale
- **Distributions**: contract_rebate_pct=NORMAL(0.35,0.15), net_price=ZIPF(1.7), lives_covered=ZIPF(1.6), market_share=NORMAL(0.12,0.08), drug_id=ZIPF(1.1)
- **Dim Sizes**: dim_drug=50, dim_payer=80
- **Seasonality**: formulary_review(Jan+Jul, +2.0x decisions), contract_renewal(Q4, +1.5x), launch_access(first 6 months, +3.0x activity)

## Public Sector > Grant & Program Performance Management
- **Fact**: `fact_grant_metric` (metric_id INT, grant_id INT FK→dim_grant, date_id INT FK→dim_date, expenditure NUMBER(10,2), budget_remaining NUMBER(10,2), milestone_completed_flag BOOLEAN, outcome_score NUMBER(5,2), participants_served INT)
- **Dims**: `dim_date`, `dim_grant`(grant_id, program_name, agency, awardee, start_date, end_date, total_budget), `dim_outcome`(outcome_id, name, category, target_value)
- **Grain**: One row per grant per reporting period (monthly)
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: expenditure=ZIPF(1.8), outcome_score=NORMAL(68,18), participants_served=ZIPF(1.7), grant_id=UNIFORM, milestone_completed_flag=UNIFORM(0.3)
- **Dim Sizes**: dim_grant=300, dim_outcome=40
- **Seasonality**: fiscal_year_end(Sep for federal, Jun for state, +2.0x spending), quarterly_reporting(+1.5x), new_fiscal_year(Oct, +1.3x new awards)

## Public Sector > Permitting & Inspection Analytics
- **Fact**: `fact_permit_event` (event_id INT, permit_id INT FK→dim_permit, inspector_id INT FK→dim_inspector, date_id INT FK→dim_date, event_type VARCHAR, processing_days INT, fee_amount NUMBER(8,2), violation_count INT, pass_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_permit`(permit_id, type, category, property_id, applicant_id, valuation), `dim_inspector`(inspector_id, name, certification, district, workload_capacity)
- **Grain**: One row per permit lifecycle event (application, review, inspection, approval)
- **Volume**: ~120K rows/year at medium scale
- **Distributions**: processing_days=NORMAL(18,12), fee_amount=ZIPF(1.7), violation_count=ZIPF(2.5), permit_id=UNIFORM, inspector_id=ZIPF(1.1), pass_flag=UNIFORM(0.78)
- **Dim Sizes**: dim_permit=2000, dim_inspector=50
- **Seasonality**: spring_construction(Mar-May, +1.5x), summer_peak(Jun-Aug, +1.3x), winter_slow(Dec-Feb, 0.7x), year_end_rush(Dec, +1.2x deadline)

## Public Sector > Budget Forecasting & Revenue Optimization
- **Fact**: `fact_revenue_collection` (collection_id INT, revenue_source_id INT FK→dim_revenue_source, fund_id INT FK→dim_fund, date_id INT FK→dim_date, budgeted_amount NUMBER(12,2), actual_amount NUMBER(12,2), variance_pct NUMBER(5,3), delinquency_rate NUMBER(5,3))
- **Dims**: `dim_date`, `dim_revenue_source`(source_id, name, type, tax_base, collection_method), `dim_fund`(fund_id, name, type, department, restricted_flag)
- **Grain**: One row per revenue source per fund per month
- **Volume**: ~60K rows/year at medium scale
- **Distributions**: budgeted_amount=ZIPF(1.7), actual_amount=ZIPF(1.7), variance_pct=NORMAL(0,0.08), revenue_source_id=ZIPF(1.1), delinquency_rate=NORMAL(0.04,0.03)
- **Dim Sizes**: dim_revenue_source=50, dim_fund=30
- **Seasonality**: property_tax_due(Apr+Oct, +3.0x), sales_tax_holiday(Nov-Dec, +1.3x), fiscal_year_close(Jun/Sep, +1.5x)

## Public Sector > Infrastructure Asset Management
- **Fact**: `fact_asset_inspection` (inspection_id INT, asset_id INT FK→dim_asset, date_id INT FK→dim_date, condition_score NUMBER(5,2), remaining_life_years INT, repair_cost_est NUMBER(10,2), priority_rank INT, deficiency_count INT, work_order_flag BOOLEAN)
- **Dims**: `dim_date`, `dim_asset`(asset_id, type, material, install_year, location, replacement_cost, department), `dim_deficiency`(deficiency_id, name, severity, safety_impact_flag)
- **Grain**: One row per asset inspection
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: condition_score=NORMAL(62,20), remaining_life_years=NORMAL(15,10), repair_cost_est=ZIPF(1.8), asset_id=UNIFORM, deficiency_count=ZIPF(2.0), work_order_flag=UNIFORM(0.25)
- **Dim Sizes**: dim_asset=2000, dim_deficiency=30
- **Seasonality**: spring_assessment(Mar-May, +1.5x road/bridge), fall_prep(Sep-Oct, +1.3x), post_winter_damage(Mar, +1.4x deficiencies), capital_budget_cycle(Jul-Sep, +1.2x)

## Public Sector > Public Health Surveillance
- **Fact**: `fact_surveillance_report` (report_id INT, condition_id INT FK→dim_condition, jurisdiction_id INT FK→dim_jurisdiction, date_id INT FK→dim_date, case_count INT, hospitalized INT, deaths INT, positivity_rate NUMBER(5,3), vaccination_pct NUMBER(5,3), alert_level VARCHAR)
- **Dims**: `dim_date`, `dim_condition`(condition_id, name, category, notifiable_flag, incubation_days), `dim_jurisdiction`(jurisdiction_id, name, type, population, density, state)
- **Grain**: One row per condition per jurisdiction per week
- **Volume**: ~100K rows/year at medium scale
- **Distributions**: case_count=ZIPF(1.8), hospitalized=ZIPF(2.2), deaths=ZIPF(3.0), condition_id=ZIPF(1.1), positivity_rate=NORMAL(0.05,0.04), vaccination_pct=NORMAL(0.65,0.15)
- **Dim Sizes**: dim_condition=50, dim_jurisdiction=200
- **Seasonality**: flu_season(Oct-Mar, +2.0x respiratory), summer_vector(Jun-Aug, +1.5x mosquito-borne), holiday_surge(Dec-Jan, +1.3x respiratory), back_to_school(Sep, +1.2x)

## Public Sector > Workforce Planning & Optimization
- **Fact**: `fact_position_status` (status_id INT, position_id INT FK→dim_position, date_id INT FK→dim_date, filled_flag BOOLEAN, days_vacant INT, salary NUMBER(10,2), retirement_eligible_flag BOOLEAN, performance_rating NUMBER(3,1), turnover_risk NUMBER(5,3))
- **Dims**: `dim_date`, `dim_position`(position_id, title, classification, department, grade, essential_flag, supervisor_id), `dim_department`(dept_id, name, agency, budget, headcount_authorized)
- **Grain**: One row per position per month
- **Volume**: ~120K rows/year at medium scale
- **Distributions**: days_vacant=ZIPF(2.0), salary=NORMAL(72000,25000), performance_rating=NORMAL(3.5,0.7), position_id=UNIFORM, turnover_risk=NORMAL(0.12,0.08), retirement_eligible_flag=UNIFORM(0.15)
- **Dim Sizes**: dim_position=1000, dim_department=30
- **Seasonality**: retirement_wave(Jun-Jul, +1.5x departures), fiscal_year_hiring(Oct-Nov, +1.4x fills), summer_intern(Jun-Aug, +1.3x temp), january_resignations(Jan, +1.2x)

## Public Sector > Open Data & Transparency Reporting
- **Fact**: `fact_dataset_usage` (usage_id INT, dataset_id INT FK→dim_dataset, date_id INT FK→dim_date, downloads INT, api_calls INT, unique_users INT, quality_score NUMBER(5,2), freshness_days INT, complaint_count INT)
- **Dims**: `dim_date`, `dim_dataset`(dataset_id, name, category, department, update_frequency, format, record_count, publish_date), `dim_consumer`(consumer_id, type, organization, use_case)
- **Grain**: One row per dataset per day
- **Volume**: ~80K rows/year at medium scale
- **Distributions**: downloads=ZIPF(1.9), api_calls=ZIPF(1.7), unique_users=ZIPF(1.8), dataset_id=ZIPF(1.1), quality_score=NORMAL(78,15), freshness_days=NORMAL(7,10), complaint_count=ZIPF(3.0)
- **Dim Sizes**: dim_dataset=200, dim_consumer=500
- **Seasonality**: election_season(Sep-Nov even years, +2.0x civic data), budget_season(Mar-May, +1.5x financial), academic_semester(Sep+Jan, +1.3x research), foia_spike(event-driven, +1.5x)
