<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

<script lang="ts">
  /* global google */

  import { slide } from 'svelte/transition';

  import Expandable from '../components/Expandable.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import type { SolarPanelConfig } from '../solar';
  import Table from '../components/Table.svelte';

  /* eslint-disable @typescript-eslint/ban-ts-comment */
  // @ts-ignore
  import { GoogleCharts } from 'google-charts';
  import { findSolarConfig, showMoney, showNumber } from '../utils';
  import InputNumber from '../components/InputNumber.svelte';
  import InputPanelsCount from '../components/InputPanelsCount.svelte';
  import InputMoney from '../components/InputMoney.svelte';
  import InputPercent from '../components/InputPercent.svelte';
  import InputRatio from '../components/InputRatio.svelte';

  export let expandedSection: string;
  export let configId: number;
  export let monthlyAverageEnergyBillInput: number;
  export let energyCostPerKwhInput: number;
  export let panelCapacityWattsInput: number;
  export let dcToAcDerateInput: number;
  export let solarPanelConfigs: SolarPanelConfig[];
  export let defaultPanelCapacityWatts: number;

  const icon = 'payments';
  const title = 'Protium Quickscan Analyse';

  let costChart: HTMLElement;
  let showAdvancedSettings = false;

  // [START solar_potential_calculations]
  // Solar configuration, from buildingInsights.solarPotential.solarPanelConfigs
  let panelsCount = 20;
  let yearlyEnergyDcKwh = 12000;

  // Basic settings
  let monthlyAverageEnergyBill: number = 300;
  let energyCostPerKwh = 0.31;
  let panelCapacityWatts = 400;
  let solarIncentives: number = 7000;
  let installationCostPerWatt: number = 4.0;
  let installationLifeSpan: number = 20;

  // Advanced settings
  let dcToAcDerate = 0.85;
  let efficiencyDepreciationFactor = 0.995;
  let costIncreaseFactor = 1.022;
  let discountRate = 1.04;

  // Solar installation
  let installationSizeKw: number = (panelsCount * panelCapacityWatts) / 1000;
  let installationCostTotal: number = installationCostPerWatt * installationSizeKw * 1000;

  // Energy consumption
  let monthlyKwhEnergyConsumption: number = monthlyAverageEnergyBill / energyCostPerKwh;
  let yearlyKwhEnergyConsumption: number = monthlyKwhEnergyConsumption * 12;

  // Energy produced for installation life span
  let initialAcKwhPerYear: number = yearlyEnergyDcKwh * dcToAcDerate;
  let yearlyProductionAcKwh: number[] = [...Array(installationLifeSpan).keys()].map(
    (year) => initialAcKwhPerYear * efficiencyDepreciationFactor ** year,
  );

  // Cost with solar for installation life span
  let yearlyUtilityBillEstimates: number[] = yearlyProductionAcKwh.map(
    (yearlyKwhEnergyProduced, year) => {
      const billEnergyKwh = yearlyKwhEnergyConsumption - yearlyKwhEnergyProduced;
      const billEstimate =
        (billEnergyKwh * energyCostPerKwh * costIncreaseFactor ** year) / discountRate ** year;
      return Math.max(billEstimate, 0); // bill cannot be negative
    },
  );
  let remainingLifetimeUtilityBill: number = yearlyUtilityBillEstimates.reduce((x, y) => x + y, 0);
  let totalCostWithSolar: number =
    installationCostTotal + remainingLifetimeUtilityBill - solarIncentives;
  console.log(`Kosten met zonne-energie: €${totalCostWithSolar.toFixed(2)}`);

  // Cost without solar for installation life span
  let yearlyCostWithoutSolar: number[] = [...Array(installationLifeSpan).keys()].map(
    (year) => (monthlyAverageEnergyBill * 12 * costIncreaseFactor ** year) / discountRate ** year,
  );
  let totalCostWithoutSolar: number = yearlyCostWithoutSolar.reduce((x, y) => x + y, 0);
  console.log(`Kosten zonder zonne-energie: €${totalCostWithoutSolar.toFixed(2)}`);

  // Savings with solar for installation life span
  let savings: number = totalCostWithoutSolar - totalCostWithSolar;
  console.log(`Besparingen: €${savings.toFixed(2)} in ${installationLifeSpan} jaar`);
  // [END solar_potential_calculations]

  // Reactive calculations
  let panelCapacityRatio: number = 1.0;
  $: panelCapacityRatio = panelCapacityWattsInput / defaultPanelCapacityWatts;
  $: installationCostTotal = installationCostPerWatt * installationSizeKw * 1000;
  $: if (solarPanelConfigs[configId]) {
    installationSizeKw = (solarPanelConfigs[configId].panelsCount * panelCapacityWattsInput) / 1000;
  }
  $: monthlyKwhEnergyConsumption = monthlyAverageEnergyBillInput / energyCostPerKwhInput;
  $: yearlyKwhEnergyConsumption = monthlyKwhEnergyConsumption * 12;
  $: if (solarPanelConfigs[configId]) {
    initialAcKwhPerYear =
      solarPanelConfigs[configId].yearlyEnergyDcKwh * panelCapacityRatio * dcToAcDerateInput;
  }
  $: yearlyProductionAcKwh = [...Array(installationLifeSpan).keys()].map(
    (year) => initialAcKwhPerYear * efficiencyDepreciationFactor ** year,
  );
  $: yearlyUtilityBillEstimates = yearlyProductionAcKwh.map((yearlyKwhEnergyProduced, year) => {
    const billEnergyKwh = yearlyKwhEnergyConsumption - yearlyKwhEnergyProduced;
    const billEstimate =
      (billEnergyKwh * energyCostPerKwhInput * costIncreaseFactor ** year) / discountRate ** year;
    return Math.max(billEstimate, 0); // bill cannot be negative
  });
  $: remainingLifetimeUtilityBill = yearlyUtilityBillEstimates.reduce((x, y) => x + y, 0);
  $: totalCostWithSolar = installationCostTotal + remainingLifetimeUtilityBill - solarIncentives;
  $: yearlyCostWithoutSolar = [...Array(installationLifeSpan).keys()].map(
    (year) =>
      (monthlyAverageEnergyBillInput * 12 * costIncreaseFactor ** year) / discountRate ** year,
  );
  $: totalCostWithoutSolar = yearlyCostWithoutSolar.reduce((x, y) => x + y, 0);
  $: savings = totalCostWithoutSolar - totalCostWithSolar;

  let energyCovered: number;
  $: energyCovered = yearlyProductionAcKwh[0] / yearlyKwhEnergyConsumption;

  let breakEvenYear: number = -1;
  $: GoogleCharts.load(
    () => {
      if (!costChart) {
        return;
      }
      const year = new Date().getFullYear();

      let costWithSolar = 0;
      const cumulativeCostsWithSolar = yearlyUtilityBillEstimates.map(
        (billEstimate, i) =>
          (costWithSolar +=
            i == 0 ? billEstimate + installationCostTotal - solarIncentives : billEstimate),
      );
      let costWithoutSolar = 0;
      const cumulativeCostsWithoutSolar = yearlyCostWithoutSolar.map(
        (cost) => (costWithoutSolar += cost),
      );
      breakEvenYear = cumulativeCostsWithSolar.findIndex(
        (costWithSolar, i) => costWithSolar <= cumulativeCostsWithoutSolar[i],
      );

      const data = google.visualization.arrayToDataTable([
        ['Jaar', 'Panelen', 'Zonder panelen'],
        [year.toString(), 0, 0],
        ...cumulativeCostsWithSolar.map((_, i) => [
          (year + i + 1).toString(),
          cumulativeCostsWithSolar[i],
          cumulativeCostsWithoutSolar[i],
        ]),
      ]);

      /* eslint-disable @typescript-eslint/no-explicit-any */
      const googleCharts = google.charts as any;
      const chart = new googleCharts.Line(costChart);
      const options = googleCharts.Line.convertOptions({
        title: `Kostenanalyse voor ${installationLifeSpan} jaar`,
        width: 350,
        height: 200,
      });
      chart.draw(data, options);
    },
    { packages: ['line'] },
  );

  function updateConfig() {
    monthlyKwhEnergyConsumption = monthlyAverageEnergyBillInput / energyCostPerKwhInput;
    yearlyKwhEnergyConsumption = monthlyKwhEnergyConsumption * 12;
    panelCapacityRatio = panelCapacityWattsInput / defaultPanelCapacityWatts;
    configId = findSolarConfig(
      solarPanelConfigs,
      yearlyKwhEnergyConsumption,
      panelCapacityRatio,
      dcToAcDerateInput,
    );
  }
</script>

<Expandable
  bind:section={expandedSection}
  {icon}
  {title}
  subtitle="Deze waardes zijn slechts tijdelijke plaatshouders."
  subtitle2="Aanpassen met je eigen meetwaarden."
  secondary
>
  <div class="flex flex-col space-y-4 pt-1">
    <div class="p-4 mb-4 surface-variant outline-text rounded-lg">
      <p class="relative inline-flex items-center space-x-2">
        <md-icon class="md:w-6 w-8">info</md-icon>
        <span>
          Projecties gebruiken een
          <a
            class="primary-text"
            href="https://developers.google.com/maps/documentation/solar/calculate-costs-us"
            target="_blank"
          >
            Amerikaans financieel model
            <md-icon class="text-sm">open_in_new</md-icon>
          </a>
        </span>
      </p>
    </div>

    <InputMoney
      bind:value={monthlyAverageEnergyBillInput}
      icon="credit_card"
      label="Gemiddelde maandelijkse elektriciteitsrekening"
      onChange={updateConfig}
    />

    <div class="inline-flex items-center space-x-2">
      <div class="grow">
        <InputPanelsCount bind:configId {solarPanelConfigs} />
      </div>
      <md-icon-button role={undefined} on:click={updateConfig}>
        <md-icon>sync</md-icon>
      </md-icon-button>
    </div>

    <InputMoney
      bind:value={energyCostPerKwhInput}
      icon="paid"
      label="Energiekosten per kWh"
      onChange={updateConfig}
    />

    <InputMoney
      bind:value={solarIncentives}
      icon="redeem"
      label="Overheids subsidies en regelingen"
      onChange={updateConfig}
    />

    <InputMoney
      bind:value={installationCostPerWatt}
      icon="request_quote"
      label="Installatiekosten per Watt"
      onChange={updateConfig}
    />

    <InputNumber
      bind:value={panelCapacityWattsInput}
      icon="bolt"
      label="Paneelcapaciteit"
      suffix="Watt"
      onChange={updateConfig}
    />

    <div class="flex flex-col items-center w-full">
      <md-text-button
        trailing-icon
        role={undefined}
        on:click={() => (showAdvancedSettings = !showAdvancedSettings)}
      >
        {showAdvancedSettings ? 'Hide' : 'Show'} advanced settings
        <md-icon slot="icon">
          {showAdvancedSettings ? 'expand_less' : 'expand_more'}
        </md-icon>
      </md-text-button>
    </div>

    {#if showAdvancedSettings}
      <div class="flex flex-col space-y-4" transition:slide={{ duration: 200 }}>
        <InputNumber
          bind:value={installationLifeSpan}
          icon="date_range"
          label="Levensduur van installatie "
          suffix="jaar"
          onChange={updateConfig}
        />

        <InputPercent
          bind:value={dcToAcDerateInput}
          icon="dynamic_form"
          label="DC naar AC conversie"
          onChange={updateConfig}
        />

        <InputRatio
          bind:value={efficiencyDepreciationFactor}
          icon="trending_down"
          label="Paneelefficiëntie vermindering per jaar"
          decrease
          onChange={updateConfig}
        />

        <InputRatio
          bind:value={costIncreaseFactor}
          icon="price_change"
          label="Energiekostenstijging per jaar"
          onChange={updateConfig}
        />

        <InputRatio
          bind:value={discountRate}
          icon="local_offer"
          label="Disconteringspercentage per jaar"
          onChange={updateConfig}
        />
      </div>
    {/if}

    <div class="grid justify-items-end">
      <md-filled-tonal-button
        trailing-icon
        role={undefined}
        href="https://developers.google.com/maps/documentation/solar/calculate-costs-us"
        target="_blank"
      >
        More details
        <md-icon slot="icon">open_in_new</md-icon>
      </md-filled-tonal-button>
    </div>
  </div>
</Expandable>

<div class="absolute top-0 left-0">
  {#if expandedSection == title}
    <div class="flex flex-col space-y-2 m-2">
      <SummaryCard
        {icon}
        {title}
        rows={[
          {
            icon: 'energy_savings_leaf',
            name: 'Jaarlijkse energie',
            value: showNumber(
              (solarPanelConfigs[configId]?.yearlyEnergyDcKwh ?? 0) * panelCapacityRatio,
            ),
            units: 'kWh',
          },
          {
            icon: 'speed',
            name: 'Installatiegrootte',
            value: showNumber(installationSizeKw),
            units: 'kW',
          },
          {
            icon: 'request_quote',
            name: 'Installatiekosten',
            value: showMoney(installationCostTotal),
          },
          {
            icon: [
              'battery_0_bar',
              'battery_1_bar',
              'battery_2_bar',
              'battery_3_bar',
              'battery_4_bar',
              'battery_5_bar',
              'battery_full',
            ][Math.floor(Math.min(Math.round(energyCovered * 100) / 100, 1) * 6)],
            name: 'Energiedekking',
            value: Math.round(energyCovered * 100).toString(),
            units: '%',
          },
        ]}
      />
    </div>

    <div class="mx-2 p-4 surface on-surface-text rounded-lg shadow-lg">
      <div bind:this={costChart} />
      <div class="w-full secondary-text">
        <Table
          rows={[
            {
              icon: 'wallet',
              name: 'Kosten zonder zonne-energie',
              value: showMoney(totalCostWithoutSolar),
            },
            {
              icon: 'wb_sunny',
              name: 'Kosten met zonne-energie',
              value: showMoney(totalCostWithSolar),
            },
            {
              icon: 'savings',
              name: 'Besparingen',
              value: showMoney(savings),
            },
            {
              icon: 'balance',
              name: 'Break even',
              value:
                breakEvenYear >= 0
                  ? `${breakEvenYear + new Date().getFullYear() + 1} in ${breakEvenYear + 1}`
                  : '--',
              units: 'jaar',
            },
          ]}
        />
      </div>
    </div>
  {/if}
</div>
