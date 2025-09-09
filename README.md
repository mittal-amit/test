## Context
- The goal of this work is to build a clean analytics layer on top of the **Organizations**, **Members**, and **Events** datasets.
- This involves **identifying data quality issues, designing appropriate staging and dimension models, and testing incremental and snapshot logic** to ensure historical accuracy (e.g., for utilization metrics).

## About Data
- Database - data_warehouse
- Tables - raw.organizations, raw.members & raw.events

## Progress So Far
- ### Created staging models - 
    #### `stg_organizations.sql`
    - Added an extra column **`is_active`**, derived from the `churned_at` field:
        - `is_active = true` when `churned_at` is `null`.
        - `is_active = false` otherwise.
    - Ensured `organization_id` is not null to maintain data quality.

    #### `stg_members.sql`
    - Applied a condition to exclude invalid records by filtering out rows where `member_id` is null.
    - Kept all relevant attributes (personal details, organization, eligibility, timestamps) intact for downstream use.

    #### `stg_events.sql`
    - Extracted structured columns from the nested JSON `data` field, making the dataset easier to use in downstream transformations.
    - Renamed `timestamp` to `event_timestamp` for clarity and consistency.

- ### Created dimension models
    #### `dim_organizations`
    - Created as an **incremental table** to optimize performance and avoid full refreshes on each run.
    - The model is designed to load only new or updated records based on the `loaded_at` column.
    - This ensures that the table always reflects the latest snapshot of organizations while keeping historical records intact.

    #### `dim_member_snapshot` (SCD2 on Province)
    - Implemented as a **dbt snapshot** to track **slowly changing dimension (Type 2)** changes in the `province_of_residence`.
    - Purpose: To ensure that **historical event attribution** (e.g., utilization rate by province) is always tied to the **correct province and organization** at the time of the event.
    - Behavior:
        - When a member’s **province_of_residence** changes, the snapshot logic:
            1. **Closes the old record** by updating `valid_to` to the day before the change.
            2. **Inserts a new record** with the updated province and sets `valid_from` to the change date.
        - This preserves a **full history** of province changes.
    - Example:
        - Alice moves from **Ontario → Quebec**.
        - `dim_member_snapshot` keeps both records:
            - Ontario (valid_from = 2025-01-01, valid_to = 2025-09-09)
            - Quebec (valid_from = 2025-09-10, valid_to = 9999-12-31)

- ### Created fact models
    #### `fct_events`
    The `fct_events` table consolidates all events from the `stg_events` staging table and links them with member-level information from the `dim_member_snapshot` table. This allows:
    - Attribution of events to the **correct organization** and **province** at the time of the event.
    - Accurate calculation of metrics like **Monthly Utilization Rate** and other member-level analytics.
        
    > ⚠️ Note: Some members are associated with multiple organizations. As a result, this fact table may contain duplicate events for a single member across different organizations.

 - ### Testing
    The testing has been performed which can be found in the Notion doucment with screenshots


## Metric Calculation Challenges
- ### Members in Multiple Organization
    - **Monthly Utilization Rate is calculated based on the number of members enrolled**, having a single member associated with multiple organizations can impact the calculation. 
    - Organization-level utilization rates may be underestimated or overcounted because the denominator (members) could include duplicates

## Suggestions
- ### Multiple Programs per Organization
    - There are multiple rows with different program combinations
    - If existing programs were combined, the created_at date might remain the same while the updated_at (or loaded_at) field is updated. This approach facilitates easier analysis when integrating with other datasets.
        - `[employee_assistance, mental_health]` **and** `[primary_care, mental_health, employee_assistance]` = `[employee_assistance, mental_health, primary_care, mental_health, employee_assistance]`

- ### Organization–Member Hierarchy

- ### Event-to-Program Mapping Gap
    - One thing which is unclear to determine from the event table is which program(s) each issue belongs to, based on the member’s eligible programs and the programs the organization has subscribed to.

## Conclusion
- ### Data Quality Observations
    - Some members appear under multiple organizations, causing **duplication** in downstream fact tables (e.g., `fct_events`).
    - Organizations may have multiple program combinations over time, leading to **repeated rows** in the members table.
    - The `event` table does not explicitly indicate which program each event relates to, making program-level analysis **ambiguous**.

- ### Recommendations
    - **Map employee eligibility in the fact table** to ensure that utilization metrics reflect only those programs for which the members are actually eligible. This helps avoid overcounting events for programs a member is not part of.
    - Implement a **program-level mapping** for events, linking each event (issue) to the member’s eligible programs and the organization’s subscribed programs. This enables accurate program-specific utilization analysis.

- ### Overall
    - Despite data issues, the **ETL and modeling approach (staging → dimension → fact tables)** is robust.
    - These factors affect calculations like the **Monthly Utilization Rate**, potentially leading to **overcounting or undercounting** at the organization or province level.

## Note 
- More detailed information can be found at this link with relevant screenshots 
- https://www.notion.so/Project-Progress-and-Data-Verification-Notes-2685228f921f8022a211cd5b5ac94d17?source=copy_link