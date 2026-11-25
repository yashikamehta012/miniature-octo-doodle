"""
CampusSpend - Personal Expense & Budget Analyzer
Course: Computational Thinking and Programming (2026CSAE100)
Run: python campus_spend.py

Data files are created automatically in the same folder:
- expenses.csv
- budgets.csv
"""

import csv
from datetime import datetime
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent
EXPENSE_FILE = BASE_DIR / "expenses.csv"
BUDGET_FILE = BASE_DIR / "budgets.csv"
CATEGORIES = ("Food", "Transport", "Study", "Shopping", "Entertainment", "Other")
EXPENSE_FIELDS = ("id", "date", "category", "amount", "note")
BUDGET_FIELDS = ("month", "amount")


def ensure_files():
    """Create data files with headers if they do not exist."""
    if not EXPENSE_FILE.exists():
        with EXPENSE_FILE.open("w", newline="", encoding="utf-8") as f:
            csv.DictWriter(f, fieldnames=EXPENSE_FIELDS).writeheader()
    if not BUDGET_FILE.exists():
        with BUDGET_FILE.open("w", newline="", encoding="utf-8") as f:
            csv.DictWriter(f, fieldnames=BUDGET_FIELDS).writeheader()


def load_expenses():
    ensure_files()
    with EXPENSE_FILE.open("r", newline="", encoding="utf-8") as f:
        return list(csv.DictReader(f))


def save_expenses(records):
    with EXPENSE_FILE.open("w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=EXPENSE_FIELDS)
        writer.writeheader()
        writer.writerows(records)


def load_budgets():
    ensure_files()
    result = {}
    with BUDGET_FILE.open("r", newline="", encoding="utf-8") as f:
        for row in csv.DictReader(f):
            try:
                result[row["month"]] = float(row["amount"])
            except (ValueError, KeyError):
                continue
    return result


def save_budgets(budgets):
    with BUDGET_FILE.open("w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=BUDGET_FIELDS)
        writer.writeheader()
        for month, amount in sorted(budgets.items()):
            writer.writerow({"month": month, "amount": f"{amount:.2f}"})


def valid_date(value):
    try:
        parsed = datetime.strptime(value, "%Y-%m-%d")
        return parsed.strftime("%Y-%m-%d") == value
    except ValueError:
        return False


def get_expense_by_month(records, month):
    return [r for r in records if r.get("date", "").startswith(month)]


def add_expense():
    records = load_expenses()
    date = input("Date (YYYY-MM-DD): ").strip()
    if not valid_date(date):
        print("Invalid date. Use a real date in YYYY-MM-DD format.")
        return
    print("Categories:", ", ".join(CATEGORIES))
    category = input("Category: ").strip().title()
    if category not in CATEGORIES:
        print("Invalid category. Please choose one from the displayed list.")
        return
    try:
        amount = float(input("Amount (₹): ").strip())
        if amount <= 0:
            print("Amount must be greater than zero.")
            return
    except ValueError:
        print("Please enter a valid numeric amount.")
        return
    note = input("Note (optional): ").strip()
    next_id = max([int(r["id"]) for r in records if r.get("id", "").isdigit()] + [0]) + 1
    records.append({
        "id": str(next_id), "date": date, "category": category,
        "amount": f"{amount:.2f}", "note": note
    })
    save_expenses(records)
    print(f"Expense #{next_id} saved successfully.")


def view_expenses():
    records = load_expenses()
    if not records:
        print("No expenses recorded yet.")
        return
    print("\nID   DATE         CATEGORY          AMOUNT       NOTE")
    print("-" * 65)
    for r in records:
        print(f'{r["id"]:<4} {r["date"]:<12} {r["category"]:<17} ₹{float(r["amount"]):>9.2f}   {r["note"]}')
    print(f"Total records: {len(records)}")


def edit_expense():
    records = load_expenses()
    if not records:
        print("No expenses available to edit.")
        return
    view_expenses()
    target = input("Enter expense ID to edit: ").strip()
    for r in records:
        if r["id"] == target:
            new_note = input(f'New note [{r["note"]}]: ').strip()
            new_amount = input(f'New amount [{r["amount"]}]: ').strip()
            if new_note:
                r["note"] = new_note
            if new_amount:
                try:
                    amount = float(new_amount)
                    if amount <= 0:
                        print("Amount must be greater than zero. No changes saved.")
                        return
                    r["amount"] = f"{amount:.2f}"
                except ValueError:
                    print("Invalid amount. No changes saved.")
                    return
            save_expenses(records)
            print(f"Expense #{target} updated successfully.")
            return
    print("Expense ID not found.")


def delete_expense():
    records = load_expenses()
    target = input("Enter expense ID to delete: ").strip()
    updated = [r for r in records if r["id"] != target]
    if len(updated) == len(records):
        print("Expense ID not found.")
    else:
        save_expenses(updated)
        print(f"Expense #{target} deleted successfully.")


def set_budget():
    month = input("Month (YYYY-MM): ").strip()
    try:
        datetime.strptime(month, "%Y-%m")
    except ValueError:
        print("Invalid month. Use YYYY-MM.")
        return
    try:
        amount = float(input("Monthly budget (₹): ").strip())
        if amount <= 0:
            print("Budget must be greater than zero.")
            return
    except ValueError:
        print("Please enter a valid numeric budget.")
        return
    budgets = load_budgets()
    budgets[month] = amount
    save_budgets(budgets)
    print(f"Budget for {month} saved: ₹{amount:.2f}")


def monthly_summary():
    month = input("Month (YYYY-MM): ").strip()
    try:
        datetime.strptime(month, "%Y-%m")
    except ValueError:
        print("Invalid month. Use YYYY-MM.")
        return
    selected = get_expense_by_month(load_expenses(), month)
    if not selected:
        print(f"No expenses found for {month}.")
        return
    total = sum(float(r["amount"]) for r in selected)
    budgets = load_budgets()
    budget = budgets.get(month)
    print(f"\nMONTHLY SUMMARY — {month}")
    print(f"Number of expenses : {len(selected)}")
    print(f"Total expenditure : ₹{total:.2f}")
    if budget is None:
        print("Budget status      : No budget set for this month.")
    else:
        used = total / budget * 100
        difference = budget - total
        print(f"Monthly budget     : ₹{budget:.2f}")
        print(f"Budget used        : {used:.1f}%")
        if difference < 0:
            print(f"ALERT              : Budget exceeded by ₹{abs(difference):.2f}")
        elif used >= 80:
            print(f"WARNING            : You have used {used:.1f}% of your budget.")
            print(f"Remaining budget   : ₹{difference:.2f}")
        else:
            print(f"Remaining budget   : ₹{difference:.2f}")
            print("Status             : Within budget.")


def category_analysis():
    month = input("Month (YYYY-MM): ").strip()
    try:
        datetime.strptime(month, "%Y-%m")
    except ValueError:
        print("Invalid month. Use YYYY-MM.")
        return
    selected = get_expense_by_month(load_expenses(), month)
    if not selected:
        print(f"No expenses found for {month}.")
        return
    totals = {}
    for record in selected:
        category = record["category"]
        totals[category] = totals.get(category, 0) + float(record["amount"])
    grand_total = sum(totals.values())
    print(f"\nCATEGORY ANALYSIS — {month}")
    for category, amount in sorted(totals.items(), key=lambda item: item[1], reverse=True):
        share = amount / grand_total * 100
        print(f"{category:<18} ₹{amount:>9.2f}  ({share:>5.1f}%)")
    top_category = max(totals, key=totals.get)
    print(f"Highest spending category: {top_category} (₹{totals[top_category]:.2f})")


def generate_report():
    month = input("Month (YYYY-MM): ").strip()
    try:
        datetime.strptime(month, "%Y-%m")
    except ValueError:
        print("Invalid month. Use YYYY-MM.")
        return
    selected = get_expense_by_month(load_expenses(), month)
    if not selected:
        print(f"No expenses found for {month}.")
        return
    totals = {}
    for record in selected:
        totals[record["category"]] = totals.get(record["category"], 0) + float(record["amount"])
    total = sum(totals.values())
    budgets = load_budgets()
    budget = budgets.get(month)
    report_path = BASE_DIR / f"CampusSpend_Report_{month}.txt"
    with report_path.open("w", encoding="utf-8") as f:
        f.write(f"CAMPUSSPEND MONTHLY REPORT — {month}\n")
        f.write("=" * 42 + "\n")
        f.write(f"Number of expenses: {len(selected)}\n")
        f.write(f"Total expenditure: Rs. {total:.2f}\n\n")
        f.write("Category-wise expenditure:\n")
        for category, amount in sorted(totals.items(), key=lambda item: item[1], reverse=True):
            f.write(f"- {category}: Rs. {amount:.2f} ({amount / total * 100:.1f}%)\n")
        if budget is not None:
            f.write(f"\nMonthly budget: Rs. {budget:.2f}\n")
            f.write(f"Difference: Rs. {budget - total:.2f}\n")
    print(f"Monthly report generated: {report_path.name}")


def show_menu():
    print("\n" + "=" * 52)
    print("       CAMPUSSPEND — EXPENSE & BUDGET ANALYZER")
    print("=" * 52)
    options = [
        "Add Expense", "View All Expenses", "Edit Expense",
        "Delete Expense", "Set / Update Monthly Budget",
        "View Monthly Summary", "Category-wise Analysis",
        "Generate Monthly Report", "Exit"
    ]
    for number, option in enumerate(options, 1):
        print(f"{number}. {option}")


def main():
    ensure_files()
    actions = {
        "1": add_expense, "2": view_expenses, "3": edit_expense,
        "4": delete_expense, "5": set_budget, "6": monthly_summary,
        "7": category_analysis, "8": generate_report
    }
    while True:
        show_menu()
        choice = input("Enter your choice (1-9): ").strip()
        if choice == "9":
            print("Thank you for using CampusSpend. Goodbye!")
            break
        action = actions.get(choice)
        if action:
            action()
        else:
            print("Invalid menu choice. Please select a number from 1 to 9.")


if __name__ == "__main__":
    main()
    
