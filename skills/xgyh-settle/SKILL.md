---
name: xgyh-settle
description: Settle monthly store operations data into a store statistics workbook (门店统计表.xlsx). Use when the user provides raw monthly transaction data (purchases, expenses by date/person) and summary metrics (revenue, order counts, platform fees), or asks to generate a monthly store statistics sheet. Trigger on: 门店统计, 月结算, 月统计, 运营数据, 进货数据, store settlement, monthly report, or pasting raw transaction logs with dates like "4-1进货". When the user pastes raw transaction data with person labels and date-item-amount lines, invoke this skill immediately.
---

# 月度门店结算

## How this skill works

The user provides two things:
1. An xlsx file path (or you search the working directory for `门店统计表.xlsx` or similar)
2. Raw monthly transaction data + summary metrics

You produce: a new sheet appended to the workbook for the next month in sequence, with all formulas wired.

All formatting, styling, and structural conventions are inherited from the existing sheets. No brand-specific assumptions are hardcoded.

## Execution (follow in order)

### Step 0: Determine target month

Read the workbook's sheet names. Find sheets matching the pattern `XXXX年XX月` (e.g., `2026年03月`). Take the latest one. The new sheet is the next calendar month.

```python
import re
import openpyxl

wb = openpyxl.load_workbook(xlsx_path)
pattern = re.compile(r'(\d{4})年(\d{2})月')
existing = []
for s in wb.sheetnames:
    m = pattern.match(s)
    if m:
        existing.append((int(m.group(1)), int(m.group(2)), s))
existing.sort()
year, month, latest_name = existing[-1]

# Next month
if month == 12:
    new_year, new_month = year + 1, 1
else:
    new_year, new_month = year, month + 1
```

If no sheets match, ask the user for the year and month.

### Step 1: Copy the latest sheet as template

```python
ws = wb.copy_worksheet(wb[latest_name])
ws.title = f"{new_year}年{new_month:02d}月"
```

This preserves **everything**: merged cells, fonts, number formats, column widths, row heights, borders, fill colors. You only change cell **values**.

### Step 2: Read title pattern from template

Don't hardcode the store name. Extract the title pattern from the template sheet's B1:

```python
template_title = wb[latest_name]['B1'].value
# e.g. "某某店--3月门店统计" → "某某店--{new_month}月门店统计"
import re
new_title = re.sub(r'\d+月', f'{new_month}月', template_title)
ws['B1'] = new_title
```

### Step 3: Parse the raw transaction data

The user's data follows this structure:

```
PersonLabel:
M-D item1 amount1
         item2 amount2

AnotherPerson:
M-D item3 amount3
```

Rules:
- Lines ending with `:` → person/store label (note for context, discard otherwise)
- Lines starting with `M-D` (month-day) → transaction date + first item + amount
- Indented lines (leading spaces) → continuation items for the most recent date
- The **last number** on each line is the amount in RMB
- Descriptors like `1000个` or parenthetical notes like `(Ba)` are qualifiers, not amounts

Parse each line into: `(item_name, amount)`. Sum amounts for identical item names across all dates and persons.

### Step 4: Categorize and sum

Map each parsed item to its E-column row. **Sum amounts for items that land in the same row.**

The category structure is read from the template's D column (rows 4-19). The standard mapping is:

| Row | D label | Common item keywords |
|-----|---------|---------------------|
| E4 | 水果 | `进货` |
| E5 | 果捞物料（酸奶/辅料） | `酸奶` |
| E6 | 零食饮料 | `干货` |
| E7 | 水电费 | `水费`, `电费` |
| E8 | 果切盒 | `果切盒` |
| E9 | 礼盒/水果打包盒类 | `果蓝`, `果叉`, `垃圾袋`, `手套` |
| E10 | 贴纸/封条/海报/卡片/配件 | `水果贴标`, `外卖封口贴`, `贴标` |
| E11 | 设备(软/硬件/收银纸) | `收银机`, `系统`, `收银纸` |
| E12 | 运营(线下) | (usually 0) |
| E13 | 美团 | platform promotion (0 if not mentioned) |
| E14 | 饿了么 | platform promotion fees from summary |
| E15 | 租金 | monthly rent — **carry from previous month** if not specified |
| E16 | 人工 | labor (0 if not mentioned) |
| E17 | 其他 | miscellaneous (0 if not mentioned) |
| E18 | 油费 | `油费` |
| E19 | 保养/其他 | `垃圾费`, `停车费`, `灯带` |

**If the template's D column labels differ**, use the template's labels — the above is the common case. The mapping logic stays the same: match item keywords to the closest D label.

**Important:** E15 (租金) defaults to the previous month's E15 value unless the user explicitly provides a different number. Never silently set it to 0.

If an item doesn't clearly match any row, ask the user.

### Step 5: Extract summary metrics

The user provides these at the end of their data:

```
线下总营业额：60740.5
线下总单数：1142
美团营业额：10455.89
饿了么营业额：10510.57
饿了么推广费：1416.63
```

Also watch for variants like `美团推广费`, `线上总营业额`, or platform-specific labels. Extract each value. If any are missing, ask.

### Step 6: Write values and formulas

**A. Revenue (row 3):**
```python
ws['B3'] = offline_revenue
ws['C3'] = meituan_rev + eleme_rev  # online total
```

**B. Cost amounts (E4-E19):**
Write each summed amount as a hardcoded number. These are the source of truth from raw data.

**C. Auto-carry values from previous month:**
```python
ws_prev = wb[latest_name]
# Carry rent forward if not specified
if ws['E15'].value == 0 or ws['E15'].value is None:
    ws['E15'] = ws_prev['E15'].value
# Previous month's order count
ws['F21'] = ws_prev['F20'].value
```

**D. F column (占比明细) — rows 4-19:**
```python
for row in range(4, 20):
    ws[f'F{row}'] = f'=E{row}/$F$24'
```

**E. G column (百分比类目) — one per category block:**
```python
ws['G4']  = '=SUM($E$4:$E$6)/$F$24'
ws['G7']  = '=SUM($E$7:$E$10)/$F$24'
ws['G11'] = '=SUM($E$11:$E$12)/$F$24'
ws['G13'] = '=SUM($E$13:$E$14)/$F$24'
ws['G15'] = '=SUM($E$15:$E$17)/$F$24'
ws['G18'] = '=SUM($E$18:$E$19)/$F$24'
```

The row ranges must match the merged cell blocks in the template. Verify these ranges against the template's merged cells before writing.

**F. 利润统算 (rows 20-30):**
```python
ws['F20'] = order_count

ws['F22'] = '=SUM(E4:E6)'
ws['F23'] = '=SUM(E7:E19)'
ws['F24'] = '=F22+F23'
ws['G22'] = '=F22/F24'
ws['G23'] = '=F23/F24'
ws['G24'] = '=F24/F24'

ws['F25'] = offline_revenue
ws['F26'] = meituan_rev
ws['F27'] = eleme_rev
# E28:F28 is merged — write formula to E28 (top-left), NOT F28
ws['E28'] = '=F25+F26+F27'
ws['G25'] = '=F25/E28'
ws['G26'] = '=F26/E28'
ws['G27'] = '=F27/E28'
ws['G28'] = '=E28/E28'

ws['E29'] = '=E28-F22'
ws['G29'] = '=E29/E28'
ws['E30'] = '=E28-F24'
ws['G30'] = '=E30/E28'
```

**G. Fix red font on formula-derived totals:**
```python
from openpyxl.styles import Font
red_font = Font(name='黑体', size=12, bold=False, color='FFFF0000')
for ref in ['F22', 'F23', 'F24', 'G22', 'G23', 'G24', 'G25', 'G26', 'G27', 'G28']:
    ws[ref].font = red_font
black_font = Font(name='黑体', size=12, bold=False)
ws['G29'].font = black_font
ws['G30'].font = black_font
```

### Step 7: Verify

Run these checks programmatically. **Do not skip.**

```python
# 1. Zero F28 references (merged cell trap)
for row in range(1, ws.max_row + 1):
    for col in range(1, ws.max_column + 1):
        cell = ws.cell(row=row, column=col)
        if cell.value and str(cell.value).startswith('=') and 'F28' in str(cell.value):
            raise ValueError(f"{cell.coordinate} references F28 — change to E28")

# 2. Math sanity check
direct = sum(ws[f'E{r}'].value or 0 for r in [4, 5, 6])
other  = sum(ws[f'E{r}'].value or 0 for r in range(7, 20))
total_cost = direct + other
total_rev = ws['B3'].value + ws['C3'].value
gross = total_rev - direct
net = total_rev - total_cost
print(f"直接成本={direct:,.2f}  其他成本={other:,.2f}  总成本={total_cost:,.2f}")
print(f"总营业额={total_rev:,.2f}  毛利润={gross:,.2f}  净利润={net:,.2f}")
print(f"净利润率={net/total_rev*100:.1f}%")

# 3. Merged cells match template
template_merges = set(str(m) for m in wb[latest_name].merged_cells.ranges)
new_merges = set(str(m) for m in ws.merged_cells.ranges)
assert template_merges == new_merges, f"Merged cell diff: {template_merges ^ new_merges}"
```

### Step 8: Save and report

```python
wb.save(xlsx_path)
```

Present a compact summary:

```
## {new_year}年{new_month:02d}月 — 已生成

| 类别 | 金额 |
|------|------|
| 直接成本 | ¥xx,xxx |
| 其他成本 | ¥x,xxx |
| 总成本 | ¥xx,xxx |
| 总营业额 | ¥xx,xxx |
| 净利润 | ¥xx,xxx (xx.x%) |
```

List any auto-carried values (e.g., "租金已从上月结转 ¥7,000") and items you had to ask about.

## Critical Rules

1. **F28 is poison.** E28:F28 is merged with E28 as top-left. All formulas reference `E28`, never `F28`.
2. **Rent carries forward.** E15 defaults to the previous month's value unless the user overrides. Never silently zero it.
3. **上月订单数 (F21) comes from the previous month's F20.** Read it, don't guess.
4. **Title auto-detected from template.** Use `re.sub` to swap the month number in B1, don't hardcode the store name.
5. **Sheet name auto-incremented.** Find the latest `XXXX年XX月` sheet and create the next calendar month.
6. **Use formulas, not hardcoded calculations.** F column, G column, and 利润统算 rows 22-30 are all formulas.
7. **Copy the latest sheet as template.** Don't build formatting from scratch.
8. **Run Step 7 verification before reporting success.** Every time.
