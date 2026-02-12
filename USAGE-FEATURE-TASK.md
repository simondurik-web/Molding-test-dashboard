# Task: Add Inventory Usage Tracking & Depletion Forecast

## File: index.html (14332 lines)

## Overview
Add 7-day and 30-day usage averages, days-until-minimum, days-until-zero calculations, and projection lines on the inventory history chart. Only for "Purchased" items.

## Step 1: Read Make/Purchased/Com column in loadInventoryData() (line ~5284)

In `loadInventoryData()`, the `prodTotalsMap` is built around line 5302. Add reading of column "Make/Purchased/Com":

After line `drawing2Url: drawing2Url` (around line 5310), add:
```js
itemType: (row['Make/Purchased/Com'] || '').toString().trim(),
```

Then in the `mergedData.push()` calls (around lines 5340 and 5358), add `itemType` field:
```js
itemType: prodData.itemType || ''
```

## Step 2: Add usage calculation function (after getPartMinimum around line 9166)

Add this new function:

```js
function calculateUsageStats(partNumber) {
    const partData = inventoryHistoryData.find(p => p.partNumber === partNumber);
    if (!partData || Object.keys(partData.dataByDate).length < 2) {
        return { usage7: null, usage30: null, daysToMin: null, daysToZero: null };
    }
    
    // Sort all dates chronologically
    const allDates = Object.keys(partData.dataByDate).map(dateStr => {
        const parts = dateStr.split('/');
        return { str: dateStr, date: new Date(parts[2], parts[0] - 1, parts[1]) };
    }).sort((a, b) => a.date - b.date);
    
    if (allDates.length < 2) return { usage7: null, usage30: null, daysToMin: null, daysToZero: null };
    
    const latest = allDates[allDates.length - 1];
    const currentQty = partData.dataByDate[latest.str];
    
    // Calculate usage over N days: find the data point closest to N days ago
    function getUsageRate(days) {
        const targetDate = new Date(latest.date);
        targetDate.setDate(targetDate.getDate() - days);
        
        // Find closest data point to target date
        let closest = null;
        let closestDiff = Infinity;
        for (const d of allDates) {
            const diff = Math.abs(d.date - targetDate);
            if (diff < closestDiff) {
                closestDiff = diff;
                closest = d;
            }
        }
        
        if (!closest || closest.str === latest.str) return null;
        
        const pastQty = partData.dataByDate[closest.str];
        const actualDays = (latest.date - closest.date) / (1000 * 60 * 60 * 24);
        if (actualDays < 1) return null;
        
        // Usage = how much inventory decreased per day (positive = consuming)
        const dailyUsage = (pastQty - currentQty) / actualDays;
        return dailyUsage > 0 ? dailyUsage : 0; // If inventory went up, usage is 0 (restocked)
    }
    
    const usage7 = getUsageRate(7);
    const usage30 = getUsageRate(30);
    
    // Use 30-day rate for projections, fall back to 7-day
    const projectionRate = usage30 !== null && usage30 > 0 ? usage30 : (usage7 !== null && usage7 > 0 ? usage7 : null);
    
    const minimum = getPartMinimum(partNumber);
    let daysToMin = null;
    let daysToZero = null;
    
    if (projectionRate && projectionRate > 0) {
        daysToZero = Math.round(currentQty / projectionRate);
        if (minimum > 0 && currentQty > minimum) {
            daysToMin = Math.round((currentQty - minimum) / projectionRate);
        } else if (minimum > 0 && currentQty <= minimum) {
            daysToMin = 0; // Already below minimum
        }
    }
    
    return { usage7, usage30, daysToMin, daysToZero, projectionRate, currentQty };
}
```

## Step 3: Add new stat boxes to the Inventory History Modal HTML (around line 1930)

After the existing `historyStatsPanel` div (which has Current, Minimum, Maximum, Average - ends around line 1934), add a NEW stats row for usage data. Insert AFTER the closing `</div>` of `historyStatsPanel`:

```html
<div class="history-stats usage-stats" id="historyUsagePanel" style="display:none;">
    <div class="history-stat">
        <div class="history-stat-value" id="historyUsage7">-</div>
        <div class="history-stat-label">7-Day Avg Usage<br><small>/day</small></div>
    </div>
    <div class="history-stat">
        <div class="history-stat-value" id="historyUsage30">-</div>
        <div class="history-stat-label">30-Day Avg Usage<br><small>/day</small></div>
    </div>
    <div class="history-stat">
        <div class="history-stat-value" id="historyDaysToMin">-</div>
        <div class="history-stat-label">Days to Minimum</div>
    </div>
    <div class="history-stat">
        <div class="history-stat-value" id="historyDaysToZero">-</div>
        <div class="history-stat-label">Days to Zero</div>
    </div>
    <div class="history-stat">
        <div class="history-stat-value" id="historyUsageTrend">-</div>
        <div class="history-stat-label">Usage Trend</div>
    </div>
</div>
```

## Step 4: Update renderInventoryHistoryChart() to show usage stats and projection lines

In `renderInventoryHistoryChart()` (starts around line 8843), after the stats update block (around line 8909 where it sets historyStatAvg), add:

```js
// Calculate and display usage stats (only for Purchased items)
const invItem = inventoryData.find(i => i.partNumber === currentHistoryPart);
const isPurchased = invItem && invItem.itemType === 'Purchased';
const usagePanel = document.getElementById('historyUsagePanel');

if (isPurchased) {
    const usage = calculateUsageStats(currentHistoryPart);
    usagePanel.style.display = 'grid';
    
    document.getElementById('historyUsage7').textContent = usage.usage7 !== null ? usage.usage7.toFixed(1) : 'N/A';
    document.getElementById('historyUsage30').textContent = usage.usage30 !== null ? usage.usage30.toFixed(1) : 'N/A';
    
    if (usage.daysToMin !== null) {
        document.getElementById('historyDaysToMin').textContent = usage.daysToMin === 0 ? '⚠️ NOW' : usage.daysToMin + 'd';
        document.getElementById('historyDaysToMin').style.color = usage.daysToMin < 14 ? 'var(--danger)' : (usage.daysToMin < 30 ? 'var(--warning)' : 'var(--success)');
    } else {
        document.getElementById('historyDaysToMin').textContent = 'N/A';
        document.getElementById('historyDaysToMin').style.color = 'var(--text-secondary)';
    }
    
    if (usage.daysToZero !== null) {
        document.getElementById('historyDaysToZero').textContent = usage.daysToZero + 'd';
        document.getElementById('historyDaysToZero').style.color = usage.daysToZero < 30 ? 'var(--danger)' : (usage.daysToZero < 60 ? 'var(--warning)' : 'var(--success)');
    } else {
        document.getElementById('historyDaysToZero').textContent = 'N/A';
        document.getElementById('historyDaysToZero').style.color = 'var(--text-secondary)';
    }
    
    // Trend: compare 7-day vs 30-day
    if (usage.usage7 !== null && usage.usage30 !== null && usage.usage30 > 0) {
        const ratio = usage.usage7 / usage.usage30;
        if (ratio > 1.15) {
            document.getElementById('historyUsageTrend').textContent = '📈 Increasing';
            document.getElementById('historyUsageTrend').style.color = 'var(--danger)';
        } else if (ratio < 0.85) {
            document.getElementById('historyUsageTrend').textContent = '📉 Decreasing';
            document.getElementById('historyUsageTrend').style.color = 'var(--success)';
        } else {
            document.getElementById('historyUsageTrend').textContent = '➡️ Stable';
            document.getElementById('historyUsageTrend').style.color = 'var(--text-secondary)';
        }
    } else {
        document.getElementById('historyUsageTrend').textContent = '-';
        document.getElementById('historyUsageTrend').style.color = 'var(--text-secondary)';
    }
} else {
    usagePanel.style.display = 'none';
}
```

## Step 5: Add projection lines to the chart

In `renderInventoryHistoryChart()`, where the Chart.js config is built (around line 8960), we need to add projection datasets. After the main `dataset` is defined and BEFORE the `charts.inventoryHistory = new Chart(ctx, {` call:

```js
// Build projection datasets for purchased items
const extraDatasets = [];
if (isPurchased) {
    const usage = calculateUsageStats(currentHistoryPart);
    if (usage.projectionRate && usage.projectionRate > 0 && usage.currentQty > 0) {
        // Add projected future data points (extend chart into the future)
        const projectionDays = Math.min(Math.ceil(usage.currentQty / usage.projectionRate) + 5, 120); // max 120 days
        const projectionLabels = [];
        const projectionData = [];
        const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
        
        // Start from current (last actual data point)
        const lastActualDate = new Date();
        for (let d = 0; d <= projectionDays; d++) {
            const futureDate = new Date(lastActualDate);
            futureDate.setDate(futureDate.getDate() + d);
            const projQty = Math.max(0, usage.currentQty - (usage.projectionRate * d));
            
            if (d === 0) {
                // First projection point = last actual point (connect them)
                projectionLabels.push(labels[labels.length - 1]);
            } else {
                projectionLabels.push(`${monthNames[futureDate.getMonth()]} ${futureDate.getDate()}`);
            }
            projectionData.push(projQty);
            if (projQty <= 0) break; // Stop at zero
        }
        
        // Pad actual data with nulls for projection length, and pad projection data with nulls for actual length
        const paddedProjection = new Array(dataPoints.length - 1).fill(null).concat(projectionData);
        const allLabels = labels.concat(projectionLabels.slice(1)); // skip first projection label (duplicate of last actual)
        
        // Pad actual data to match extended labels
        while (dataPoints.length < allLabels.length) dataPoints.push(null);
        
        // Replace labels array
        labels.length = 0;
        allLabels.forEach(l => labels.push(l));
        
        // Projection line (30-day rate)
        extraDatasets.push({
            label: 'Projected (30d avg)',
            data: paddedProjection,
            borderColor: '#e53e3e',
            backgroundColor: 'rgba(229, 62, 62, 0.05)',
            borderDash: [8, 4],
            fill: false,
            tension: 0,
            pointRadius: 0,
            pointHoverRadius: 4,
            borderWidth: 2
        });
        
        // Add minimum threshold line across full range if minimum > 0
        if (minimum > 0) {
            extraDatasets.push({
                label: 'Minimum Threshold',
                data: new Array(labels.length).fill(minimum),
                borderColor: '#ed8936',
                borderDash: [4, 4],
                fill: false,
                pointRadius: 0,
                pointHoverRadius: 0,
                borderWidth: 1.5
            });
        }
        
        // Zero line
        extraDatasets.push({
            label: 'Zero',
            data: new Array(labels.length).fill(0),
            borderColor: 'rgba(229, 62, 62, 0.3)',
            borderDash: [2, 2],
            fill: false,
            pointRadius: 0,
            pointHoverRadius: 0,
            borderWidth: 1
        });
        
        // Update suggestedMax to include projection
        suggestedMin = 0;
    }
}
```

Then in the Chart creation, change `datasets: [dataset]` to:
```js
datasets: [dataset, ...extraDatasets]
```

And enable the legend when there are extra datasets:
```js
legend: { display: extraDatasets.length > 0, labels: { color: '#a0aec0', usePointStyle: true } },
```

## Step 6: Add CSS for the usage stats panel

In the CSS section (around line 666-880 where the inventory history styles are), add:

```css
.usage-stats {
    border-top: 1px solid rgba(255,255,255,0.1);
    padding-top: 12px;
    margin-top: 8px;
}
.usage-stats .history-stat-label small {
    font-size: 10px;
    color: var(--text-secondary);
    opacity: 0.7;
}
```

## IMPORTANT NOTES
- The `labels` array and `dataPoints` array are modified in-place in Step 5 (projection lines extend them). This is intentional — the chart needs to show future dates.
- Only show usage stats and projection for items where `itemType === 'Purchased'`. For Make items, hide the usage panel.
- The `inventoryHistoryData` must be loaded before `calculateUsageStats` works — it's already loaded in `openInventoryHistoryModal` via `await loadInventoryHistoryData()`.
- The version number is currently v7.30.2 — update to v7.31.0 (search for the version string).
