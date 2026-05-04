# Alteryx ETL for Order Processing and Data Warehouse Updates

This repository contains an Alteryx workflow designed to automate a real-world Extract, Transform, and Load (ETL) process for a wholesale organization. The primary goal is to update a central data warehouse with new monthly sales and customer information from a Point of Sale (POS) system.

## Project Overview

As an analyst for a wholesale company, my task was to create a repeatable Alteryx workflow to process the order data for August 2021. This involves integrating new sales data with returns information and updating both the historical orders table and the master customer details table in our data warehouse.

This workflow demonstrates key data preparation, blending, and transformation techniques in Alteryx to ensure data integrity and accuracy before loading it into the final destination tables.

## Core Objectives

1.  **Update the Orders Table:**
    *   Integrate new POS orders from August 2021 into the existing data warehouse `Orders` table.
    *   Merge returns data for August, correctly flagging orders that were returned. This includes updating both new August orders and existing historical orders that were returned in August.
2.  **Update the Customers Table:**
    *   Refresh the master `Customers` table with the most recent customer details (e.g., address) captured during their latest purchase in August.
    *   Ensure data consistency by converting full state names to their standard two-letter codes.

## Technical Details

*   **Platform:** The entire workflow was built and executed using **Alteryx Designer**.
*   **Data Environment:** To simulate a real-world scenario without requiring a database connection, this project uses local files as substitutes for database tables:
    *   **Microsoft Excel (.xlsx)** for the historical `Orders` and `Customers` tables.
    *   **Comma-Separated Values (.csv)** for the new `POS Orders` and `Returns Data` feeds.

## Setup and Usage

To run this project, you will need Alteryx Designer installed. The repository includes all necessary input files and the starter workflow.

1.  **Open the Workflow:** Navigate to the project directory and open the **`Starter Workflow 12.2.yxmd`** file in Alteryx Designer.
2.  **Review the Inputs:** The workflow canvas is pre-populated with all the required `Input Data` tools, which are configured to read from the provided Excel and CSV files.
3.  **Run the Workflow:** Click the **Run** button (or press `Ctrl+R`) in Alteryx Designer to execute the entire process.
4.  **Check the Outputs:** Upon successful completion, the workflow will generate two new files in the same directory:
    *   `output-orders.xlsx`
    *   `output-customers.xlsx`

## Data Sources

The workflow utilizes the following data inputs to simulate the company's data ecosystem:

| Data Source                | Description                                                                                                                   | Format      |
| :------------------------- | :---------------------------------------------------------------------------------------------------------------------------- | :---------- |
| **POS Orders**             | Transactional data for orders placed in August 2021.                                                                          | CSV         |
| **Data Warehouse Orders**  | The historical table of all orders placed before August 2021.                                                                 | Excel       |
| **Data Warehouse Customers** | The master table of all registered customers.                                                                                 | Excel       |
| **Returns Data**           | A list of `OrderID`s for items returned during August 2021.                                                                   | CSV         |
| **US State Codes**         | A reference list for mapping full state names to two-letter abbreviations.                                                     | Text Input  |

## Workflow Breakdown & Transformations

The workflow is logically split into two main streams, each targeting one of the final output tables.

### Part 1: Updating the Orders Table

This stream prepares and combines all historical and new order data, ensuring the `Returned` status is accurate.

1.  **Data Type Correction:** A `Select` tool is used on all inputs to standardize data types (e.g., ensuring `OrderID` is a string, `Price` is a number, and `OrderDate` is a date format).
2.  **Isolate POS Order Details:** Customer-specific columns are removed from the POS data stream using a `Select` tool to match the schema of the target `Orders` table, which only contains a `CustomerID`.
3.  **Process New August Orders:**
    *   A `Join` tool merges the August POS data with the `Returns Data` on `OrderID`.
    *   **Not Returned:** Orders from the Left output (not found in the returns list) have a new `Returned` column added via a `Formula` tool, with the value set to `False`.
    *   **Returned:** Orders from the Join output (found in the returns list) already contain the `Returned` flag from the returns data.
4.  **Process Historical Orders:**
    *   Another `Join` tool merges the existing `Data Warehouse Orders` data with the remaining returns (Right output from the previous join), which correspond to orders placed before August.
    *   The `Select` tool is used on the Join output to resolve the duplicated `Returned` column, keeping the updated `True` value from the returns data and renaming it to match the original column name.
5.  **Final Union:** A `Union` tool combines all four streams into a single, comprehensive dataset:
    *   New August orders that were not returned.
    *   New August orders that were returned.
    *   Historical orders that were returned in August (status updated).
    *   Historical orders that were not returned in August (status unchanged).
6.  **Output:** The consolidated data is written to the **`output-orders.xlsx`** file.

### Part 2: Updating the Customers Table

This stream focuses on refreshing the master customer list with the most recent information.

1.  **Isolate Customer Details:** A `Select` tool is used on the August POS data to remove order-specific details (product, quantity, etc.), keeping only the customer fields. The `CustomerID` field is renamed to `ID` to match the master table.
2.  **Handle Duplicate Customers:**
    *   **Challenge:** The POS data contains multiple records for a single customer if they purchased multiple items. This would incorrectly duplicate records in the master table.
    *   **Solution:** A `Unique` tool is applied to the data, keeping only one record per unique `ID` (Customer ID). This ensures we only use the latest customer information once.
3.  **Standardize State Codes:** A `Find and Replace` tool is used to transform the full state names from the POS data into two-letter codes, using the `US State Codes` reference table.
4.  **Merge with Existing Customers:**
    *   A `Join` tool merges the cleaned, unique customer data from the POS system (Left input) with the `Data Warehouse Customers` table (Right input) on `ID`.
    *   **Updated Customers:** The Join output contains customers who placed an order in August. A `Select` tool is used to drop the old, redundant address columns from the right input, effectively keeping the new data from the left.
    *   **Unchanged Customers:** The Right output contains customers who did not place an order in August. Their data remains as-is.
5.  **Final Union:** A `Union` tool combines the updated customer records and the unchanged customer records.
6.  **Output:** The final, complete master customer list is written to the **`output-customers.xlsx`** file.

## Validation

To verify that the workflow has run correctly, you can compare the output files you generated against the provided solution files included in the project resources. If your `output-orders.xlsx` and `output-customers.xlsx` files are identical to the examples, the workflow is functioning as intended.

## Key Alteryx Tools & Concepts

*   **Input/Output:** `Input Data`, `Output Data`, `Text Input`
*   **Preparation:** `Select` (for data types, renaming, and removing columns), `Formula`
*   **Join:** `Join` (to combine data on common keys), `Union` (to append datasets)
*   **Transform:** `Find and Replace`, `Unique`
*   **Documentation:** `Comment`, `Tool Container` (to organize the workflow)
*   **Development:** Caching and running workflows for efficient testing.

## Challenges & Considerations

*   **Data Type Mismatch:** A common issue when joining data from different sources (like CSVs and Excel files). This was proactively managed by using `Select` tools to ensure key fields were the same data type before any join operation.
*   **File Overwriting:** The Output Data tool was configured to **"Overwrite File (Remove)"** to prevent errors from locked files and ensure a fresh output is generated on each workflow run.
*   **Workflow Readability:** Containers and annotations were used extensively to group logical steps and document the purpose of each section, making the complex workflow easy to understand and maintain.

## Key Learnings

This project highlights several core principles of working effectively in Alteryx, as emphasized by the instructor:

*   **Flexibility is a Feature:** Alteryx allows for many different approaches to solve the same problem. There isn't one "perfect" way; the platform enables you to build a solution that matches your own thought process.
*   **Think in Small Steps:** Breaking down a complex problem into smaller, manageable steps (often one tool at a time) is a powerful strategy. This makes the logic easier to build, test, and debug.
*   **Organization is Crucial:** As workflows grow, their complexity can make them difficult to understand. Using `Tool Containers` to group logical sections and adding clear `Annotations` to tools and connections is essential for creating maintainable and shareable workflows.
*   **Iterate Efficiently:** Using the **`Cache and Run Workflow`** feature on large inputs dramatically speeds up the development cycle, allowing for rapid testing and iteration without repeatedly loading the same data. 
