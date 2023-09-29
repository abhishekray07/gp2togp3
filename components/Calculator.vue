<template>
  <div class="min-h-screen flex flex-col items-center bg-gray-100 py-5">
    <h1 class="text-2xl font-bold mb-4">
      AWS EBS gp2 to gp3 Savings Calculator
    </h1>

    <p class="font-semibold text-normal mb-4">Current gp2 Volume</p>

    <form
      class="w-1/3 sm:w-1/2"
      v-on:submit.prevent="calculateSavings"
      method="POST"
      action="#"
    >
      <div class="bg-white p-5 rounded-lg shadow-md">
        <div class="mb-4 mt-4">
          <label for="gp2Size" class="block text-sm font-medium text-gray-600"
            >Volume Size (in GB):</label
          >
          <input
            type="number"
            id="gp2Size"
            v-model="gp2Size"
            class="mt-1 p-2 w-full border rounded-md"
          />
        </div>
        <div class="mb-4 mt-4">
          <label for="gp2Size" class="block text-sm font-medium text-gray-600"
            >Baseline IOPS Per Month</label
          >
          <input
            type="number"
            id="gp2IOPS"
            v-model="gp2IOPS"
            class="mt-1 p-2 w-full border rounded-md"
            disabled
          />
        </div>
        <div class="mb-4 mt-4">
          <label for="gp2Size" class="block text-sm font-medium text-gray-600"
            >Baseline Throughput Per Month (MiB/s):</label
          >
          <input
            type="number"
            id="gp2Throughput"
            v-model="gp2Throughput"
            class="mt-1 p-2 w-full border rounded-md"
            disabled
          />
        </div>
      </div>

      <p class="font-semibold text-normal mt-6 mb-2">Equivalent gp3 Volume</p>
      <div class="bg-white p-5 rounded-lg shadow-md mt-2">
        <div class="mb-4">
          <label for="gp3Size" class="block text-sm font-medium text-gray-600"
            >Volume Size (in GB):</label
          >
          <input
            type="number"
            id="gp3Size"
            v-model="gp3Size"
            class="mt-1 p-2 w-full border rounded-md"
          />
        </div>

        <div class="mb-4">
          <label for="gp3IOPS" class="block text-sm font-medium text-gray-600"
            >Provisioned IOPS:</label
          >
          <input
            type="number"
            id="gp3IOPS"
            v-model="gp3IOPS"
            class="mt-1 p-2 w-full border rounded-md"
            min="3000"
            max="16000"
          />
        </div>

        <div class="mb-4">
          <label
            for="gp3Throughput"
            class="block text-sm font-medium text-gray-600"
            >Provisioned Throughput(in MiB/s):</label
          >
          <input
            type="number"
            id="gp3Throughput"
            v-model="gp3Throughput"
            class="mt-1 p-2 w-full border rounded-md"
            min="125"
            max="1000"
          />
        </div>
      </div>
      <button
        class="w-full bg-blue-500 hover:bg-blue-600 text-white font-medium p-2 rounded mb-4 mt-4"
      >
        Calculate Savings
      </button>
    </form>
    <div class="mt-5 text-lg">
      <p>
        <strong>Estimated Monthly Savings:</strong> ${{ savings.toFixed(2) }}
      </p>
      <p>
        <strong>Percentage Reduction in Price:</strong>
        {{ reductionPercentage.toFixed(2) }}%
      </p>
    </div>
    <div class="mt-5 w-full max-w-2xl">
      <table class="min-w-full divide-y divide-gray-200">
        <thead>
          <tr class="bg-gray-50">
            <th class="px-4 py-2 border">Volume Type</th>
            <th class="px-4 py-2 border">gp3</th>
            <th class="px-4 py-2 border">gp2</th>
          </tr>
        </thead>
        <tbody class="bg-white divide-y divide-gray-200">
          <tr>
            <td class="px-4 py-2 border font-semibold">Volume Size</td>
            <td class="px-4 py-2 border">1 GiB - 16 TiB</td>
            <td class="px-4 py-2 border">1 GiB - 16 TiB</td>
          </tr>
          <tr>
            <td class="px-4 py-2 border font-semibold">Baseline IOPS</td>
            <td class="px-4 py-2 border">3000</td>
            <td class="px-4 py-2 border">
              3 IOPS/GiB (minimum 100 IOPS) to a maximum of 16,000 IOPS. <br />
            </td>
          </tr>
          <tr>
            <td class="px-4 py-2 border font-semibold">Max IOPS</td>
            <td class="px-4 py-2 border">16,000</td>
            <td class="px-4 py-2 border">16,000 <br /></td>
          </tr>

          <tr>
            <td class="px-4 py-2 border font-semibold">Price</td>
            <td class="px-4 py-2 border">
              $0.08/GiB-month <br />

              3,000 IOPS free and <br />

              $0.005/provisioned IOPS-month over 3,000; <br />

              125 MiB/s free and <br />

              $0.04/provisioned MiB/s-month over 125MiB/s <br />
            </td>
            <td class="px-4 py-2 border">$0.10/GiB-month<br /></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from "vue";

const gp2Size = ref(1000);
const gp2IOPS = ref(3000);
const gp2Throughput = ref(250);
const gp3Size = ref(1000);
const gp3IOPS = ref(3000);
const gp3Throughput = ref(250);

const savings = ref(0);
const reductionPercentage = ref(0);

const gp2MonthlyCost = ref(0);
const gp3StorageCost = ref(0);
const gp3IOPSCost = ref(0);
const gp3ThroughputCost = ref(0);
const gp3MonthlyCost = ref(0);

const calculateSavings = () => {
  const gp2PricePerGB = 0.1;
  const gp3PricePerGB = 0.08;
  const gp3ExtraIOPSPrice = 0.005; // per IOPS over 3000
  const gp3ExtraThroughputPrice = 0.04; // per MB/s over 125

  gp2MonthlyCost.value = gp2Size.value * gp2PricePerGB;
  gp3StorageCost.value = gp3Size.value * gp3PricePerGB;
  const extraIOPS = Math.max(0, gp3IOPS.value - 3000);
  gp3IOPSCost.value = extraIOPS * gp3ExtraIOPSPrice;
  const extraThroughput = Math.max(0, gp3Throughput.value - 125);
  gp3ThroughputCost.value = extraThroughput * gp3ExtraThroughputPrice;

  gp3MonthlyCost.value =
    gp3StorageCost.value + gp3IOPSCost.value + gp3ThroughputCost.value;
  savings.value = gp2MonthlyCost.value - gp3MonthlyCost.value;
  reductionPercentage.value = (savings.value / gp2MonthlyCost.value) * 100;
};

watch(gp2Size, () => {
  gp3Size.value = gp2Size.value;
  gp2IOPS.value = Math.min(16000, Math.max(100, gp2Size.value * 3));
  gp2Throughput.value = Math.min(250, gp2Size.value * 0.125);
});

watch(gp2IOPS, () => {
  gp3IOPS.value = gp2IOPS.value;
});

watch(gp2Throughput, () => {
  gp3Throughput.value = gp2Throughput.value;
});
</script>

  <style scoped>
/* You can add any additional styles here, if needed */
</style>