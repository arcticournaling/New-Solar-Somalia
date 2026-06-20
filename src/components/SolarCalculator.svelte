<script lang="ts">
  import { onDestroy } from "svelte";

  type GeneratorUse = "none" | "occasional" | "daily";
  type BatteryChoice = "yes" | "no";
  type AnswerValue = string | number;
  type PanelOption = {
    watts: number;
    area: number;
    label: string;
  };

  type Answers = {
    businessName: string;
    businessType: string;
    district: string;
    monthlyKwh: number;
    electricityCostPerKwh: number;
    roofArea: number;
    coveragePercentage: number;
    generatorUse: GeneratorUse;
    monthlyDieselCost: number;
    solarBudget: number;
    batteryBackup: BatteryChoice;
  };

  type Results = {
    dailyKwh: number;
    idealSystemSizeKw: number;
    idealPanelCount: number;
    idealRoofAreaRequired: number;
    maxPanelsByRoof: number;
    maxSystemSizeKw: number;
    effectiveSystemSizeKw: number;
    inverterSizeKw: number;
    isRoofConstrained: boolean;
    monthlySolarProduction: number;
    monthlyGridSavings: number;
    dieselSavings: number;
    totalMonthlySavings: number;
    solarCostLow: number;
    solarCostHigh: number;
    batteryCostLow: number;
    batteryCostHigh: number;
    softCostLow: number;
    softCostHigh: number;
    estimatedCostLow: number;
    estimatedCostHigh: number;
    simplePaybackYearsLow: number;
    simplePaybackYearsHigh: number;
    batteryStorageKwh: number;
    batteryModuleCount: number;
    reliabilityNote: string;
    businessNote: string;
    budgetMessage: string;
  };

  const PEAK_SUN_HOURS = 5.7;
  const SYSTEM_EFFICIENCY = 0.75;
  const PANEL_POWER_WATTS = 550;
  const ROOF_AREA_PER_PANEL = 2.6;
  const SOLAR_COST_LOW_PER_KW = 1000;
  const SOLAR_COST_HIGH_PER_KW = 1600;
  const BATTERY_COST_LOW_PER_KWH = 300;
  const BATTERY_COST_HIGH_PER_KWH = 500;
  const BATTERY_MODULE_SIZE_KWH = 5;

  const panelOptions: PanelOption[] = [
    { watts: 270, area: 1.6, label: "270 W common panel" },
    { watts: 450, area: 2.1, label: "450 W modern panel" },
    { watts: 550, area: 2.6, label: "550 W commercial panel" },
  ];

  const businessTypes = ["Shop", "Office", "Restaurant", "Clinic", "Warehouse", "Hotel", "Other"];
  const districts = [
    "Abdiaziz",
    "Bondhere",
    "Daynile",
    "Dharkenley",
    "Hamar Jajab",
    "Hamar Weyne",
    "Hodan",
    "Howlwadaag",
    "Karaan",
    "Shangani",
    "Shibis",
    "Waberi",
    "Wadajir",
    "Wardhigley",
    "Yaqshid",
  ];

  const questions = [
    "What is the business name?",
    "Where in Mogadishu is it located?",
    "How much electricity do you use each month?",
    "What is your electricity cost per kWh?",
    "How much open roof space do you have? (m²)",
    "How much of your electricity should solar cover?",
    "Do you use a diesel generator?",
    "How much do you spend on diesel each month?",
    "What is your budget for solar?",
    "Do you want battery backup?",
  ];

  const defaultAnswers: Answers = {
    businessName: "",
    businessType: "Shop",
    district: "Hodan",
    monthlyKwh: 1200,
    electricityCostPerKwh: 0.6,
    roofArea: 80,
    coveragePercentage: 75,
    generatorUse: "occasional",
    monthlyDieselCost: 150,
    solarBudget: 0,
    batteryBackup: "yes",
  };

  let currentStep = 0;
  let showResults = false;
  let savedMessage = "";
  let validationMessage = "";
  let activeHelp = "";
  let selectedPanelWatts = PANEL_POWER_WATTS;
  let answers: Answers = { ...defaultAnswers };

  function updateAnswer(key: keyof Answers, value: AnswerValue) {
    answers = {
      ...answers,
      [key]: value,
    };
    savedMessage = "";
    validationMessage = "";
  }

  function toPositiveNumber(value: number) {
    return Number.isFinite(value) && value > 0 ? value : 0;
  }

  function toZeroOrPositiveNumber(value: number) {
    return Number.isFinite(value) && value > 0 ? value : 0;
  }

  function roundToHalf(value: number) {
    return Math.ceil(value * 2) / 2;
  }

  function calculateResults(currentAnswers: Answers): Results {
    const monthlyKwh = toPositiveNumber(currentAnswers.monthlyKwh);
    const electricityCostPerKwh = toPositiveNumber(currentAnswers.electricityCostPerKwh);
    const monthlyDieselCost = toZeroOrPositiveNumber(currentAnswers.monthlyDieselCost);
    const solarBudget = toZeroOrPositiveNumber(currentAnswers.solarBudget);
    const roofArea = toPositiveNumber(currentAnswers.roofArea);

    // Formula: daily_kwh = monthly_kwh / 30
    const dailyKwh = monthlyKwh / 30;

    // Formula: coverage_decimal = coverage_percentage / 100
    const coverageDecimal = toPositiveNumber(currentAnswers.coveragePercentage) / 100;

    // Formula: target_daily_solar = daily_kwh * coverage_decimal
    const targetDailySolar = dailyKwh * coverageDecimal;

    // Formula: system_size_kw = target_daily_solar / (5.7 * 0.75)
    const idealSystemSizeKw = targetDailySolar / (PEAK_SUN_HOURS * SYSTEM_EFFICIENCY);

    // Formula: panel_count = ceil((system_size_kw * 1000) / 550)
    const idealPanelCount = Math.ceil((idealSystemSizeKw * 1000) / PANEL_POWER_WATTS);

    // Formula: roof_area_required = panel_count * 2.6
    const idealRoofAreaRequired = idealPanelCount * ROOF_AREA_PER_PANEL;

    // Formula: max_panels_by_roof = floor(roof_area / 2.6)
    const maxPanelsByRoof = Math.floor(roofArea / ROOF_AREA_PER_PANEL);

    // Formula: max_system_size_kw = (max_panels_by_roof * 550) / 1000
    const maxSystemSizeKw = (maxPanelsByRoof * PANEL_POWER_WATTS) / 1000;

    const isRoofConstrained = idealPanelCount > maxPanelsByRoof;
    const effectiveSystemSizeKw = isRoofConstrained ? maxSystemSizeKw : idealSystemSizeKw;
    const inverterSizeKw = roundToHalf(effectiveSystemSizeKw * 1.15);

    // Roof-limited production and savings use the effective system size.
    const monthlySolarProduction = effectiveSystemSizeKw * PEAK_SUN_HOURS * 30 * SYSTEM_EFFICIENCY;
    const monthlyGridSavings = Math.min(monthlySolarProduction, monthlyKwh) * electricityCostPerKwh;

    // Formula: diesel_savings = 0, 25%, or 60% of monthly diesel cost.
    const dieselSavings =
      currentAnswers.generatorUse === "daily"
        ? monthlyDieselCost * 0.6
        : currentAnswers.generatorUse === "occasional"
          ? monthlyDieselCost * 0.25
          : 0;

    // Formula: total_monthly_savings = monthly_grid_savings + diesel_savings
    const totalMonthlySavings = monthlyGridSavings + dieselSavings;

    // Conservative solar-only cost range.
    const solarCostLow = effectiveSystemSizeKw * SOLAR_COST_LOW_PER_KW;
    const solarCostHigh = effectiveSystemSizeKw * SOLAR_COST_HIGH_PER_KW;

    // Battery storage is total energy capacity. It may be made from multiple modules.
    const batteryStorageKwh = currentAnswers.batteryBackup === "yes" ? dailyKwh * 0.3 : 0;
    const batteryModuleCount =
      currentAnswers.batteryBackup === "yes" ? Math.ceil(batteryStorageKwh / BATTERY_MODULE_SIZE_KWH) : 0;
    const batteryCostLow = batteryStorageKwh * BATTERY_COST_LOW_PER_KWH;
    const batteryCostHigh = batteryStorageKwh * BATTERY_COST_HIGH_PER_KWH;

    // Soft costs include mounting, wiring, protection equipment, transport,
    // installation labor, and small design/site costs.
    const softCostLow = solarCostLow * 0.15;
    const softCostHigh = solarCostHigh * 0.25;
    const estimatedCostLow = solarCostLow + batteryCostLow + softCostLow;
    const estimatedCostHigh = solarCostHigh + batteryCostHigh + softCostHigh;

    // Formula: simple_payback_years = estimated_cost / (total_monthly_savings * 12)
    const simplePaybackYearsLow = totalMonthlySavings > 0 ? estimatedCostLow / (totalMonthlySavings * 12) : 0;
    const simplePaybackYearsHigh = totalMonthlySavings > 0 ? estimatedCostHigh / (totalMonthlySavings * 12) : 0;

    return {
      dailyKwh,
      idealSystemSizeKw,
      idealPanelCount,
      idealRoofAreaRequired,
      maxPanelsByRoof,
      maxSystemSizeKw,
      effectiveSystemSizeKw,
      inverterSizeKw,
      isRoofConstrained,
      monthlySolarProduction,
      monthlyGridSavings,
      dieselSavings,
      totalMonthlySavings,
      solarCostLow,
      solarCostHigh,
      batteryCostLow,
      batteryCostHigh,
      softCostLow,
      softCostHigh,
      estimatedCostLow,
      estimatedCostHigh,
      simplePaybackYearsLow,
      simplePaybackYearsHigh,
      batteryStorageKwh,
      batteryModuleCount,
      reliabilityNote: getReliabilityNote(currentAnswers.generatorUse, currentAnswers.batteryBackup),
      businessNote: getBusinessNote(currentAnswers.businessType),
      budgetMessage: getBudgetMessage(solarBudget, estimatedCostLow, estimatedCostHigh),
    };
  }

  $: results = calculateResults(answers);

  $: annualSavings = results.totalMonthlySavings * 12;
  $: businessLabel = answers.businessType === "Other" ? "business" : answers.businessType.toLowerCase();
  $: businessSummaryLine = `For ${answers.businessName.trim()} ${businessLabel} in ${answers.district}.`;
  $: selectedPanel = panelOptions.find((option) => option.watts === selectedPanelWatts) || panelOptions[2];
  $: displayedPanelCount = Math.ceil((results.effectiveSystemSizeKw * 1000) / selectedPanel.watts);
  $: displayedRoofSpace = displayedPanelCount * selectedPanel.area;

  $: summaryText =
    results.totalMonthlySavings > 0
      ? `You could save about ${formatCurrency(results.totalMonthlySavings)} per month, or about ${formatCurrency(annualSavings)} per year. ` +
        `After about ${formatPaybackRange(results)}, those savings may have paid back the estimated system cost. ` +
        `From that point, more of this money can stay in the business instead of going to grid and generator costs.`
      : "We need savings data to estimate when the system may pay for itself.";

  $: if (typeof document !== "undefined") {
    document.body.classList.toggle("calculator-results-active", showResults);
  }

  onDestroy(() => {
    if (typeof document !== "undefined") {
      document.body.classList.remove("calculator-results-active");
    }
  });

  function getBudgetMessage(solarBudget: number, estimatedCostLow: number, estimatedCostHigh: number) {
    if (solarBudget <= 0) {
      return "";
    }

    if (solarBudget < estimatedCostLow) {
      return "Your budget may be too low for this system.";
    }

    if (solarBudget <= estimatedCostHigh) {
      return "Your budget may cover a lower-cost version, but quotes should be checked.";
    }

    return "Your budget may be enough for this estimated system.";
  }

  function toggleHelp(key: string) {
    activeHelp = activeHelp === key ? "" : key;
  }

  function closeHelp() {
    activeHelp = "";
  }

  function handleEscape(event: KeyboardEvent) {
    if (event.key === "Escape") {
      closeHelp();
    }
  }

  function getBusinessNote(businessType: string) {
    if (businessType === "Hotel") {
      return "Hotels often have heavy cooling, laundry, refrigeration, and evening loads. Battery backup may be important.";
    }

    if (businessType === "Shop") {
      return "Shops may benefit most from daytime solar if most usage happens during business hours.";
    }

    if (businessType === "Clinic") {
      return "Clinics should prioritize reliability and backup for critical equipment.";
    }

    if (businessType === "Restaurant") {
      return "Restaurants often need backup for refrigeration and evening operations.";
    }

    if (businessType === "Office") {
      return "Offices usually benefit strongly from daytime solar coverage.";
    }

    return "Review the business load profile before final design, especially equipment that runs after sunset.";
  }

  function getReliabilityNote(generatorUse: GeneratorUse, batteryBackup: BatteryChoice) {
    if (generatorUse === "daily" && batteryBackup === "yes") {
      return "Daily generator users should combine solar with battery backup for critical loads and confirm the design with a site engineer.";
    }

    if (generatorUse === "daily") {
      return "Solar can reduce generator fuel use, but without batteries the business may still need generator support during outages.";
    }

    if (generatorUse === "occasional" && batteryBackup === "yes") {
      return "Battery backup can help cover short outages and reduce occasional generator runtime.";
    }

    if (generatorUse === "occasional") {
      return "Solar should reduce grid purchases, while occasional generator use may still be needed for reliability.";
    }

    if (batteryBackup === "yes") {
      return "Battery backup adds resilience for selected loads even if generator use is currently low.";
    }

    return "This setup is focused on lowering bills. Add battery backup later if outage protection becomes important.";
  }

  function validateStep(step: number) {
    if (step === 0 && answers.businessName.trim().length === 0) {
      return "Business name is required.";
    }

    if (step === 2 && toPositiveNumber(answers.monthlyKwh) <= 0) {
      return "Monthly kWh must be greater than 0.";
    }

    if (step === 3 && toPositiveNumber(answers.electricityCostPerKwh) <= 0) {
      return "Electricity cost per kWh must be greater than 0.";
    }

    if (step === 3 && answers.electricityCostPerKwh > 2) {
      return "Electricity cost per kWh must be $2 or less.";
    }

    if (step === 4 && toPositiveNumber(answers.roofArea) <= 0) {
      return "Roof area must be greater than 0.";
    }

    if (step === 7 && (!Number.isFinite(answers.monthlyDieselCost) || answers.monthlyDieselCost < 0)) {
      return "Monthly diesel cost can be 0, but it cannot be negative.";
    }

    if (step === 8 && (!Number.isFinite(answers.solarBudget) || answers.solarBudget < 0)) {
      return "Solar budget can be 0, but it cannot be negative.";
    }

    return "";
  }

  function goNext() {
    const message = validateStep(currentStep);
    if (message) {
      validationMessage = message;
      return;
    }

    if (currentStep === questions.length - 1) {
      showResults = true;
      savedMessage = "Estimate ready.";
      return;
    }

    currentStep += 1;
  }

  function showFinalResults() {
    const message = validateStep(currentStep);
    if (message) {
      validationMessage = message;
      return;
    }

    showResults = true;
    savedMessage = "Estimate ready.";
  }

  function finishWithBatteryChoice(value: BatteryChoice) {
    answers = {
      ...answers,
      batteryBackup: value,
    };
    validationMessage = "";
    savedMessage = "";
  }

  function goBack() {
    validationMessage = "";

    if (showResults) {
      showResults = false;
      currentStep = questions.length - 1;
      return;
    }

    currentStep = Math.max(0, currentStep - 1);
  }

  function startOver() {
    currentStep = 0;
    showResults = false;
    savedMessage = "";
    validationMessage = "";
    activeHelp = "";
    selectedPanelWatts = PANEL_POWER_WATTS;
    answers = { ...defaultAnswers };
  }

  function saveSubmission() {
    const submission = {
      answers: { ...answers },
      results,
      createdAt: new Date().toISOString(),
    };

    const existingSubmissions = JSON.parse(localStorage.getItem("newSolarSubmissions") || "[]");
    existingSubmissions.push(submission);
    localStorage.setItem("newSolarSubmissions", JSON.stringify(existingSubmissions));
    savedMessage = "Estimate saved in this browser.";
  }

  function escapeHtml(value: string) {
    return value
      .replaceAll("&", "&amp;")
      .replaceAll("<", "&lt;")
      .replaceAll(">", "&gt;")
      .replaceAll('"', "&quot;")
      .replaceAll("'", "&#039;");
  }

  function reportRow(label: string, value: string) {
    return `
      <div class="report-row">
        <span>${escapeHtml(label)}</span>
        <strong>${escapeHtml(value)}</strong>
      </div>
    `;
  }

  function openPdfReport() {
    const batteryText =
      results.batteryStorageKwh > 0
        ? `${formatNumber(results.batteryStorageKwh)} kWh storage, about ${results.batteryModuleCount} modules of ${BATTERY_MODULE_SIZE_KWH} kWh`
        : "Not selected";

    const reportWindow = window.open("", "_blank");

    if (!reportWindow) {
      savedMessage = "Allow popups to open the PDF report.";
      return;
    }

    reportWindow.document.write(`
      <!doctype html>
      <html lang="en">
        <head>
          <meta charset="UTF-8" />
          <meta name="viewport" content="width=device-width, initial-scale=1.0" />
          <title>Business Solar Estimate</title>
          <style>
            :root {
              color: #173025;
              font-family: Inter, Arial, sans-serif;
            }

            body {
              margin: 0;
              background: #f7ecd6;
            }

            main {
              max-width: 760px;
              margin: 0 auto;
              padding: 32px;
              background: #ffffff;
            }

            h1,
            h2 {
              margin: 0;
              color: #1b5e20;
              font-family: Manrope, Inter, Arial, sans-serif;
            }

            h1 {
              font-size: 32px;
              line-height: 1.1;
            }

            h2 {
              margin-top: 28px;
              padding-top: 18px;
              border-top: 1px solid #e2d1b4;
              font-size: 18px;
            }

            p {
              margin: 8px 0 0;
              color: #5d6d61;
            }

            .report-grid {
              display: grid;
              gap: 10px;
              margin-top: 14px;
            }

            .report-row {
              display: grid;
              grid-template-columns: 1fr 1.4fr;
              gap: 16px;
              padding: 10px 0;
              border-bottom: 1px solid #ead8b8;
            }

            .report-row span {
              color: #5d6d61;
              font-weight: 700;
            }

            .report-row strong {
              color: #173025;
              font-weight: 800;
            }

            ul {
              margin: 12px 0 0;
              padding-left: 20px;
              color: #173025;
            }

            li {
              margin: 6px 0;
            }

            .disclaimer {
              margin-top: 12px;
              padding: 12px;
              background: #fff8e8;
              border: 1px solid #e2d1b4;
              border-radius: 6px;
              color: #173025;
            }

            @media print {
              body {
                background: #ffffff;
              }

              main {
                padding: 0;
              }
            }
          </style>
        </head>
        <body>
          <main>
            <h1>Business Solar Estimate</h1>
            <p>${escapeHtml(businessSummaryLine)}</p>

            <h2>Location</h2>
            <p>${escapeHtml(answers.district)}, Mogadishu, Somalia</p>

            <h2>Recommended System</h2>
            <div class="report-grid">
              ${reportRow("Recommended System", `${formatNumber(results.effectiveSystemSizeKw)} kW`)}
              ${reportRow("Panels", `${displayedPanelCount} x ${selectedPanel.watts} W panels`)}
              ${reportRow("Roof Space", `${formatNumber(displayedRoofSpace)} m²`)}
              ${reportRow("Estimated Production", `${formatNumber(results.monthlySolarProduction, 0)} kWh per month`)}
              ${reportRow("Estimated Cost", formatCurrencyRange(results.estimatedCostLow, results.estimatedCostHigh))}
              ${reportRow("Estimated Savings", `${formatCurrency(results.totalMonthlySavings)} per month / ${formatCurrency(annualSavings)} per year`)}
              ${reportRow("Simple Payback", formatPaybackRange(results))}
            </div>

            <h2>Assumptions</h2>
            <ul>
              <li>5.7 peak sun hours/day</li>
              <li>75% system efficiency</li>
              <li>Cost estimates based on commercial installations</li>
            </ul>

            <h2>Recommended equipment class</h2>
            <div class="report-grid">
              ${reportRow("Panels", `${selectedPanel.label}`)}
              ${reportRow("Inverter", `Commercial inverter, about ${formatNumber(results.inverterSizeKw)} kW`)}
              ${reportRow("Battery", batteryText)}
              ${reportRow("Mounting", "Commercial roof mounting for flat or low-slope roofs")}
            </div>

            <h2>Disclaimer</h2>
            <p class="disclaimer">Preliminary estimate only. Final system design requires a site visit and engineering review.</p>
          </main>
          <script>
            window.addEventListener("load", () => {
              window.print();
            });
          <\/script>
        </body>
      </html>
    `);
    reportWindow.document.close();
    savedMessage = "PDF report opened. Choose Save as PDF in the print dialog.";
  }

  function formatNumber(value: number, digits = 1) {
    return new Intl.NumberFormat("en-US", {
      maximumFractionDigits: digits,
      minimumFractionDigits: digits,
    }).format(Number.isFinite(value) ? value : 0);
  }

  function formatCurrency(value: number) {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: "USD",
      maximumFractionDigits: 0,
    }).format(Number.isFinite(value) ? value : 0);
  }

  function formatCurrencyRange(low: number, high: number) {
    return `${formatCurrency(low)}–${formatCurrency(high)}`;
  }

  function formatPaybackRange(currentResults: Results) {
    if (currentResults.totalMonthlySavings <= 0) {
      return "Not enough savings data";
    }

    return `${formatNumber(currentResults.simplePaybackYearsLow)}–${formatNumber(currentResults.simplePaybackYearsHigh)} years`;
  }
</script>

<svelte:window on:click={closeHelp} on:keydown={handleEscape} />

<section class="wizard-shell" aria-label="Solar calculator wizard">
  {#if !showResults}
    <form class="wizard-card" on:submit|preventDefault={goNext}>
      <div class="progress-row">
        <p class="eyebrow">Step {currentStep + 1} of {questions.length}</p>
        <div class="progress-track" aria-hidden="true">
          <span style={`width: ${((currentStep + 1) / questions.length) * 100}%`}></span>
        </div>
      </div>

      <h2>{questions[currentStep]}</h2>

      {#if currentStep === 0}
        <div class="step-fields">
          <label>
            <span>Business name</span>
            <input
              autocomplete="organization"
              type="text"
              value={answers.businessName}
              on:input={(event) => updateAnswer("businessName", event.currentTarget.value)}
            />
          </label>
          <div>
            <span class="field-label">Business type</span>
            <div class="option-list">
              {#each businessTypes as type}
                <button
                  class:active-option={answers.businessType === type}
                  class="option-button"
                  type="button"
                  on:click={() => updateAnswer("businessType", type)}
                >
                  {type}
                </button>
              {/each}
            </div>
          </div>
        </div>
      {:else if currentStep === 1}
        <label>
          <span>District / area</span>
          <select
            value={answers.district}
            on:change={(event) => updateAnswer("district", event.currentTarget.value)}
          >
            {#each districts as district}
              <option value={district}>{district}</option>
            {/each}
          </select>
        </label>
      {:else if currentStep === 2}
        <label>
          <div class="label-with-help">
            <span>Monthly kWh</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "monthly-kwh"} aria-label="Help for monthly kWh" type="button" on:click={() => toggleHelp("monthly-kwh")}>?</button>
              {#if activeHelp === "monthly-kwh"}
                <span class="help-copy">Your business electricity use for one month. You may find this on your electricity meter or bill.</span>
              {/if}
            </span>
          </div>
          <input
            max="2"
            min="0"
            step="any"
            type="number"
            value={answers.monthlyKwh}
            on:input={(event) => updateAnswer("monthlyKwh", Number(event.currentTarget.value))}
          />
        </label>
      {:else if currentStep === 3}
        <label>
          <span>Electricity cost per kWh</span>
          <input
            min="0"
            step="any"
            type="number"
            value={answers.electricityCostPerKwh}
            on:input={(event) => updateAnswer("electricityCostPerKwh", Number(event.currentTarget.value))}
          />
        </label>
      {:else if currentStep === 4}
        <label>
          <div class="label-with-help">
            <span>Roof area</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "roof-area"} aria-label="Help for roof area" type="button" on:click={() => toggleHelp("roof-area")}>?</button>
              {#if activeHelp === "roof-area"}
                <span class="help-copy">The flat open roof space where panels can be placed. Do not include shaded areas.</span>
              {/if}
            </span>
          </div>
          <input
            min="0"
            step="any"
            type="number"
            value={answers.roofArea}
            on:input={(event) => updateAnswer("roofArea", Number(event.currentTarget.value))}
          />
          <small class="helper-text">If unsure, estimate the flat, unshaded area available for panels.</small>
        </label>
      {:else if currentStep === 5}
        <div>
          <div class="label-with-help">
            <span>Solar coverage</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "solar-coverage"} aria-label="Help for solar coverage" type="button" on:click={() => toggleHelp("solar-coverage")}>?</button>
              {#if activeHelp === "solar-coverage"}
                <span class="help-copy">How much of your electricity you want solar to cover. 100% means solar makes about the same energy you use in one month.</span>
              {/if}
            </span>
          </div>
          <div class="option-list compact-options">
            {#each [50, 75, 100, 120] as coverage}
              <button
                class:active-option={answers.coveragePercentage === coverage}
                class="option-button"
                type="button"
                on:click={() => updateAnswer("coveragePercentage", coverage)}
              >
                {coverage}%
              </button>
            {/each}
          </div>
        </div>
      {:else if currentStep === 6}
        <div class="option-list compact-options">
          <button
            class:active-option={answers.generatorUse === "none"}
            class="option-button"
            type="button"
            on:click={() => updateAnswer("generatorUse", "none")}
          >
            None
          </button>
          <button
            class:active-option={answers.generatorUse === "occasional"}
            class="option-button"
            type="button"
            on:click={() => updateAnswer("generatorUse", "occasional")}
          >
            Occasional
          </button>
          <button
            class:active-option={answers.generatorUse === "daily"}
            class="option-button"
            type="button"
            on:click={() => updateAnswer("generatorUse", "daily")}
          >
            Daily
          </button>
        </div>
      {:else if currentStep === 7}
        <label>
          <span>Diesel cost per month</span>
          <input
            min="0"
            step="any"
            type="number"
            value={answers.monthlyDieselCost}
            on:input={(event) => updateAnswer("monthlyDieselCost", Number(event.currentTarget.value))}
          />
        </label>
      {:else if currentStep === 8}
        <label>
          <span>Solar budget</span>
          <input
            min="0"
            step="any"
            type="number"
            value={answers.solarBudget}
            on:input={(event) => updateAnswer("solarBudget", Number(event.currentTarget.value))}
          />
          <small class="helper-text">Enter 0 if you do not have a budget yet.</small>
        </label>
      {:else if currentStep === 9}
        <div class="option-list compact-options">
          <button
            class:active-option={answers.batteryBackup === "yes"}
            class="option-button"
            type="button"
            on:click={() => finishWithBatteryChoice("yes")}
          >
            Yes
          </button>
          <button
            class:active-option={answers.batteryBackup === "no"}
            class="option-button"
            type="button"
            on:click={() => finishWithBatteryChoice("no")}
          >
            No
          </button>
          <small class="helper-text">Choose an option, then show results.</small>
        </div>
      {/if}

      {#if validationMessage}
        <p class="field-error" role="alert">{validationMessage}</p>
      {/if}

      <div class="wizard-actions">
        <button class="button secondary" disabled={currentStep === 0} type="button" on:click={goBack}>
          Back
        </button>
        {#if currentStep === questions.length - 1}
          <button class="button primary" type="button" on:click={showFinalResults}>Show results</button>
        {:else}
          <button class="button primary" type="button" on:click={goNext}>Next</button>
        {/if}
      </div>
    </form>
  {/if}

  <section
    class:hidden-results={!showResults}
    class="results-panel"
    aria-labelledby="results-heading"
  >
      <div class="results-heading">
        <p class="eyebrow">Results</p>
        <h2 id="results-heading">Solar estimate</h2>
        <p>{businessSummaryLine}</p>
      </div>

      {#if results.isRoofConstrained}
        <article class="warning-card">
          <h3>Roof warning</h3>
          <p>Your roof may not fit the full system needed for your selected coverage.</p>
        </article>
      {/if}

      {#if answers.solarBudget > 0}
        <article class="warning-card">
          <h3>{answers.solarBudget < results.estimatedCostLow ? "Budget warning" : "Budget comparison"}</h3>
          <p><strong>User budget:</strong> {formatCurrency(answers.solarBudget)}</p>
          <p><strong>Estimated total cost range:</strong> {formatCurrencyRange(results.estimatedCostLow, results.estimatedCostHigh)}</p>
          <p>{results.budgetMessage}</p>
        </article>
      {/if}

      <article class="summary-card estimate-summary">
        <h3>Plain-English summary</h3>
        <p>{summaryText}</p>
      </article>

      <div class="result-grid">
        <article class="result-card">
          <div class="label-with-help">
            <span>System size</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "system-size"} aria-label="Help for system size" type="button" on:click={() => toggleHelp("system-size")}>?</button>
              {#if activeHelp === "system-size"}
                <span class="help-copy">The recommended size of the solar system.</span>
              {/if}
            </span>
          </div>
          <strong>{formatNumber(results.idealSystemSizeKw)} kW</strong>
        </article>
        {#if results.isRoofConstrained}
          <article class="result-card">
            <span>Roof-limited size</span>
            <strong>{formatNumber(results.maxSystemSizeKw)} kW</strong>
          </article>
        {/if}
        <article class="result-card">
          <div class="label-with-help">
            <span>Panels needed</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "panels-needed"} aria-label="Help for panels needed" type="button" on:click={() => toggleHelp("panels-needed")}>?</button>
              {#if activeHelp === "panels-needed"}
                <span class="help-copy">Approximate number of {selectedPanel.watts} W solar panels.</span>
              {/if}
            </span>
          </div>
          <strong>{displayedPanelCount}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Roof space needed (m²)</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "roof-needed"} aria-label="Help for roof space needed" type="button" on:click={() => toggleHelp("roof-needed")}>?</button>
              {#if activeHelp === "roof-needed"}
                <span class="help-copy">Estimated roof area needed for the recommended system.</span>
              {/if}
            </span>
          </div>
          <strong>{formatNumber(displayedRoofSpace)}</strong>
        </article>
        <article class="result-card">
          <span>Your roof space (m²)</span>
          <strong>{formatNumber(answers.roofArea)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Solar energy per month</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "solar-energy"} aria-label="Help for solar energy per month" type="button" on:click={() => toggleHelp("solar-energy")}>?</button>
              {#if activeHelp === "solar-energy"}
                <span class="help-copy">Estimated electricity the system may produce each month.</span>
              {/if}
            </span>
          </div>
          <strong>{formatNumber(results.monthlySolarProduction, 0)} kWh</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Grid electricity savings</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "grid-savings"} aria-label="Help for grid electricity savings" type="button" on:click={() => toggleHelp("grid-savings")}>?</button>
              {#if activeHelp === "grid-savings"}
                <span class="help-copy">Estimated monthly savings from buying less electricity.</span>
              {/if}
            </span>
          </div>
          <strong>{formatCurrency(results.monthlyGridSavings)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Diesel savings</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "diesel-savings"} aria-label="Help for diesel savings" type="button" on:click={() => toggleHelp("diesel-savings")}>?</button>
              {#if activeHelp === "diesel-savings"}
                <span class="help-copy">Estimated monthly savings from using the generator less.</span>
              {/if}
            </span>
          </div>
          <strong>{formatCurrency(results.dieselSavings)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Total monthly savings</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "total-savings"} aria-label="Help for total monthly savings" type="button" on:click={() => toggleHelp("total-savings")}>?</button>
              {#if activeHelp === "total-savings"}
                <span class="help-copy">Grid electricity savings plus diesel savings.</span>
              {/if}
            </span>
          </div>
          <strong>{formatCurrency(results.totalMonthlySavings)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Total estimated cost</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "total-cost"} aria-label="Help for total estimated cost" type="button" on:click={() => toggleHelp("total-cost")}>?</button>
              {#if activeHelp === "total-cost"}
                <span class="help-copy">Estimated cost range including solar equipment, installation, wiring, mounting, and batteries if selected.</span>
              {/if}
            </span>
          </div>
          <strong>{formatCurrencyRange(results.estimatedCostLow, results.estimatedCostHigh)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Simple payback time</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "payback"} aria-label="Help for payback period" type="button" on:click={() => toggleHelp("payback")}>?</button>
              {#if activeHelp === "payback"}
                <span class="help-copy">Estimated years for savings to equal the system cost.</span>
              {/if}
            </span>
          </div>
          <strong>{formatPaybackRange(results)}</strong>
        </article>
        <article class="result-card">
          <div class="label-with-help">
            <span>Battery storage</span>
            <span class="help-tip" on:click|stopPropagation>
              <button aria-expanded={activeHelp === "battery-storage"} aria-label="Help for battery storage" type="button" on:click={() => toggleHelp("battery-storage")}>?</button>
              {#if activeHelp === "battery-storage"}
                <span class="help-copy">Energy stored for use during outages or at night.</span>
              {/if}
            </span>
          </div>
          <strong>{results.batteryStorageKwh > 0 ? `${formatNumber(results.batteryStorageKwh)} kWh` : "Not selected"}</strong>
        </article>
        <article class="result-card panel-assumption">
          <label>
            <span>Panel rating used</span>
            <select
              value={selectedPanelWatts}
              on:change={(event) => {
                selectedPanelWatts = Number(event.currentTarget.value);
                savedMessage = "";
              }}
            >
              {#each panelOptions as option}
                <option value={option.watts}>{option.label}</option>
              {/each}
            </select>
          </label>
        </article>
      </div>

      <article class="summary-card">
        <h3>Estimated cost breakdown</h3>
        <div class="cost-breakdown">
          <div>
            <span>Solar panels, inverter, and main equipment</span>
            <strong>{formatCurrencyRange(results.solarCostLow, results.solarCostHigh)}</strong>
          </div>
          <div>
            <span>Battery storage</span>
            <strong>
              {results.batteryStorageKwh > 0
                ? formatCurrencyRange(results.batteryCostLow, results.batteryCostHigh)
                : "Not selected"}
            </strong>
          </div>
          <div>
            <span>Installation, wiring, mounting, transport, and protection equipment</span>
            <strong>{formatCurrencyRange(results.softCostLow, results.softCostHigh)}</strong>
          </div>
          <div class="cost-total">
            <span>Estimated total</span>
            <strong>{formatCurrencyRange(results.estimatedCostLow, results.estimatedCostHigh)}</strong>
          </div>
        </div>
        <p>
          This is an early estimate, not a supplier quote. Real costs can change because of shipping, import fees,
          exchange rates, roof structure, battery brand, inverter quality, and installer experience.
        </p>
      </article>

      {#if results.batteryStorageKwh > 0}
        <article class="summary-card">
          <h3>Battery setup</h3>
          <p>
            Recommended battery storage: <strong>{formatNumber(results.batteryStorageKwh)} kWh</strong>.
            Example battery setup: <strong>{results.batteryModuleCount}</strong> batteries/modules of
            <strong>{BATTERY_MODULE_SIZE_KWH} kWh</strong> each.
          </p>
          <p>Battery storage means the total energy your batteries can hold. A system can use many battery modules together.</p>
        </article>
      {/if}

      <article class="summary-card">
        <h3>Inverter setup</h3>
        <p>
          Suggested inverter size: <strong>about {formatNumber(results.inverterSizeKw)} kW</strong>.
          {answers.batteryBackup === "yes"
            ? " Ask installers for a hybrid inverter that can work with solar panels, batteries, and backup power."
            : " Ask installers for a commercial grid-tied inverter sized for the solar system, with room for site conditions and future expansion."}
        </p>
        <p>
          The final inverter choice should match the building load, phase type, battery plan, generator setup, and local electrical standards.
        </p>
      </article>

      <article class="summary-card">
        <h3>Assumptions</h3>
        <div class="assumption-list">
          <div class="assumption-item">
            <div class="label-with-help">
              <span>Panel power rating</span>
              <span class="help-tip" on:click|stopPropagation>
                <button aria-expanded={activeHelp === "panel-power"} aria-label="Help for panel power rating" type="button" on:click={() => toggleHelp("panel-power")}>?</button>
                {#if activeHelp === "panel-power"}
                  <span class="help-copy">How strong one solar panel is. A 550 W panel is common for large solar systems.</span>
                {/if}
              </span>
            </div>
            {selectedPanel.watts} W. A panel can produce up to its power rating under ideal test conditions. Real-world output is lower because of heat, dust, wires, and equipment losses.
          </div>
          <p><strong>Estimated roof area per panel:</strong> {formatNumber(selectedPanel.area)} m²</p>
          <div class="assumption-item">
            <div class="label-with-help">
              <span>System efficiency</span>
              <span class="help-tip" on:click|stopPropagation>
                <button aria-expanded={activeHelp === "system-efficiency"} aria-label="Help for system efficiency" type="button" on:click={() => toggleHelp("system-efficiency")}>?</button>
                {#if activeHelp === "system-efficiency"}
                  <span class="help-copy">Solar systems lose some energy from heat, dust, wires, and equipment. This calculator assumes 75% real-world performance.</span>
                {/if}
              </span>
            </div>
            75%
          </div>
          <div class="assumption-item">
            <div class="label-with-help">
              <span>Inverter size</span>
              <span class="help-tip" on:click|stopPropagation>
                <button aria-expanded={activeHelp === "inverter-size"} aria-label="Help for inverter size" type="button" on:click={() => toggleHelp("inverter-size")}>?</button>
                {#if activeHelp === "inverter-size"}
                  <span class="help-copy">The inverter changes solar power into electricity your business can use.</span>
                {/if}
              </span>
            </div>
            About {formatNumber(results.inverterSizeKw)} kW
          </div>
        </div>
      </article>

      {#if savedMessage}
        <p class="save-message" role="status">{savedMessage}</p>
      {/if}

      <div class="wizard-actions result-actions">
        <button class="button secondary" type="button" on:click={goBack}>Back</button>
        <button class="button secondary" type="button" on:click={openPdfReport}>PDF Report</button>
        <button class="button primary" type="button" on:click={startOver}>Start Over</button>
      </div>

      <p class="disclaimer">Estimates only. This is not a final engineering design or installation quote.</p>
  </section>
</section>
