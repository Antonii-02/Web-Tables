# Web-Tables
```
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import re
import time
driver = webdriver.Chrome()
wait = WebDriverWait(driver, 15)

try:
    # Open the demo website
    driver.get("https://assertqa.com/practice/webtables")
    driver.maximize_window()

    # Wait for the table to load
    wait.until(
        EC.presence_of_element_located((By.TAG_NAME, "table"))
    )

    # Locate the table
    table = driver.find_element(By.TAG_NAME, "table")

    # TC01: Print all column headings
    print("\nTC01 - All Column Headings")
    headers = [
        h.text.strip()
        for h in table.find_elements(By.XPATH, ".//thead//th")
        if h.text.strip()
    ]

    # Fallback if the headings are not inside thead
    if not headers:
        headers = [
            h.text.strip()
            for h in table.find_elements(By.XPATH, ".//th")
            if h.text.strip()
        ]

    print(headers)
    print("PASS" if headers else "FAIL")

    # Helper function to get visible employee rows
    def get_rows():
        current_table = driver.find_element(By.TAG_NAME, "table")
        return [
            row for row in current_table.find_elements(
                By.XPATH, ".//tbody/tr"
            )
            if row.is_displayed()
            and any(
                cell.text.strip()
                for cell in row.find_elements(By.XPATH, "./td")
            )
        ]

    # TC02: Print the first data row
    print("\nTC02 - First Data Row")
    rows = get_rows()

    if rows:
        print([cell.text.strip() for cell in rows[0].find_elements(By.TAG_NAME, "td")])
        print("PASS")
    else:
        print("FAIL - No data rows found")

    # TC03: Print the last data row across all pages
    print("\nTC03 - Last Data Row")

    # Read the number of pages from the page controls
    page_buttons = driver.find_elements(
        By.XPATH,
        "//button[normalize-space()='Last' or "
        "contains(translate(@aria-label,'LAST','last'),'last')]"
    )

    # Try a last-page button if the website provides one
    if page_buttons and page_buttons[0].is_enabled():
        page_buttons[0].click()
        time.sleep(1)

    # Generic pagination: click the Next button until disabled
    while True:
        next_buttons = driver.find_elements(
            By.XPATH,
            "//button[normalize-space()='Next' or "
            "normalize-space()='>' or "
            "contains(translate(@aria-label,'NEXT','next'),'next')]"
        )

        clickable = [
            b for b in next_buttons
            if b.is_displayed() and b.is_enabled()
        ]

        if not clickable:
            break

        clickable[0].click()
        time.sleep(0.5)

    rows = get_rows()

    if rows:
        print([cell.text.strip() for cell in rows[-1].find_elements(By.TAG_NAME, "td")])
        print("PASS")
    else:
        print("FAIL - No last row found")

    # TC04: Search employee by last name
    print("\nTC04 - Search by Last Name")
    last_name = input("Enter employee last name (example: Smith): ").strip()

    search_boxes = driver.find_elements(
        By.XPATH,
        "//input[contains(translate(@placeholder,"
        "'SEARCH','search'),'search')]"
    )

    if search_boxes:
        search_box = search_boxes[0]
        search_box.clear()
        search_box.send_keys(last_name)
        time.sleep(1)

        matches = [
            [cell.text.strip() for cell in row.find_elements(By.TAG_NAME, "td")]
            for row in get_rows()
            if last_name.lower() in row.text.lower()
        ]

        if matches:
            for record in matches:
                print(record)
            print("PASS")
        else:
            print("FAIL - No matching record")
    else:
        print("FAIL - Search box not found")

    # Clear search before extracting data
    if search_boxes:
        search_boxes[0].clear()
        time.sleep(1)

    # Collect all records across paginated pages
    print("\nTC05 - Extract All Email Addresses")
    all_records = []
    seen = set()

    while True:
        for row in get_rows():
            record = tuple(
                cell.text.strip()
                for cell in row.find_elements(By.TAG_NAME, "td")
            )
            if record and record not in seen:
                seen.add(record)
                all_records.append(record)

        next_buttons = driver.find_elements(
            By.XPATH,
            "//button[normalize-space()='Next' or "
            "normalize-space()='>' or "
            "contains(translate(@aria-label,'NEXT','next'),'next')]"
        )
        clickable = [
            b for b in next_buttons
            if b.is_displayed() and b.is_enabled()
        ]

        if not clickable:
            break

        clickable[0].click()
        time.sleep(0.5)

    emails = []
    for record in all_records:
        for value in record:
            if re.fullmatch(
                r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}",
                value
            ):
                emails.append(value)

    print(emails)
    print("PASS" if emails else "FAIL - No emails found")

    # TC06: Highest Due amount
    print("\nTC06 - Employee With Highest Due Amount")
    # This task requires a Due column. Identify it by its heading.
    due_index = next(
        (
            i for i, heading in enumerate(headers)
            if heading.strip().lower() == "due"
        ),
        None
    )

    if due_index is None:
        print("FAIL - No Due column found on this website")
    else:
        due_records = []

        for record in all_records:
            if len(record) <= due_index:
                continue

            amount_text = record[due_index].replace(",", "")
            amounts = re.findall(r"-?\d+(?:\.\d+)?", amount_text)

            if amounts:
                due_records.append((float(amounts[0]), record))

        if due_records:
            highest = max(due_records, key=lambda item: item[0])
            print("Highest Due:", highest[0])
            print("Employee:", highest[1])
            print("PASS")
        else:
            print("FAIL - No numeric Due amounts found")

    # TC07: Verify a particular website link exists
    print("\nTC07 - Verify Website Link")
    website_url = input(
        "Enter the website URL to check (example: https://example.com): "
    ).strip()

    links = driver.find_elements(By.XPATH, "//a[@href]")
    found = any(
        link.get_attribute("href").rstrip("/") == website_url.rstrip("/")
        for link in links
    )

    print("PASS - Website link exists" if found else "FAIL - Website link not found")

    # TC08: Count data rows without header
    print("\nTC08 - Count Data Rows")
    print("Total data rows collected:", len(all_records))
    print("PASS" if all_records else "FAIL - No data rows found")

finally:
    input("\nPress Enter to close the browser...")
    driver.quit()
```
