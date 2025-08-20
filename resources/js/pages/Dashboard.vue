<script setup lang="ts">
import { Head, Link } from '@inertiajs/vue3';
import { ref, onMounted } from 'vue';

// Reactive data
const reportData = ref<any>(null);
const loading = ref(true);
const error = ref<string | null>(null);

// AI Commentary state
const aiPrompt = ref('');
const aiResponse = ref('');
const aiLoading = ref(false);
const aiError = ref<string | null>(null);

// Fetch financial report data
const fetchFinancialData = async () => {
  try {
    loading.value = true;
    const response = await fetch('/api/financial-report');
    if (!response.ok) {
      throw new Error('Failed to fetch financial data');
    }
    const data = await response.json();
    reportData.value = data;
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'An error occurred';
  } finally {
    loading.value = false;
  }
};

// Generate AI commentary
const generateCommentary = async () => {
  if (!aiPrompt.value.trim()) return;
  
  try {
    aiLoading.value = true;
    aiError.value = null;
    
    const response = await fetch('/api/generate-commentary', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || ''
      },
      body: JSON.stringify({
        prompt: aiPrompt.value
      })
    });
    
    if (!response.ok) {
      throw new Error('Failed to generate commentary');
    }
    
    const data = await response.json();
    aiResponse.value = data.response || 'No response received';
  } catch (err) {
    aiError.value = err instanceof Error ? err.message : 'An error occurred';
  } finally {
    aiLoading.value = false;
  }
};

onMounted(() => {
  fetchFinancialData();
});
</script>

<template>
  <Head title="Financial Report Challenge" />
  
  <div class="min-h-screen bg-gray-50 py-8 px-4">
    <div class="max-w-7xl mx-auto">
      <!-- Header -->
      <div class="mb-8">
        <Link 
          href="/" 
          class="inline-flex items-center text-sm text-gray-600 hover:text-gray-900 mb-4"
        >
          ← Back to Home
        </Link>
        
        <h1 class="text-3xl font-bold text-gray-900 mb-2">
          🌾 Figured Financial Report Challenge
        </h1>
        <p class="text-gray-600">
          Transform this basic data display into a beautiful, interactive financial report!
        </p>
      </div>
    </div>
  </div>
</template>