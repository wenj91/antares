<template>
   <div class="container">
      <div class="columns">
         <div class="column col-16 text-right">
            <div class="mb-4">
               {{ $t('message.runQuery') }}
            </div>
            <div class="mb-4">
               {{ $t('word.format') }}
            </div>
            <div class="mb-4">
               {{ $t('word.clear') }}
            </div>
            <div class="mb-4">
               {{ $t('word.history') }}
            </div>
            <div class="mb-4">
               {{ $t('message.openNewTab') }}
            </div>
            <div class="mb-4">
               {{ $t('message.closeTab') }}
            </div>
         </div>
         <div class="column col-16">
            <div class="mb-4">
               <code>F5</code>
            </div>
            <div class="mb-4">
               <code>CTRL</code> + <code>B</code>
            </div>
            <div class="mb-4">
               <code>CTRL</code> + <code>ALT</code> + <code>W</code>
            </div>
            <div class="mb-4">
               <code>CTRL</code> + <code>G</code>
            </div>
            <div class="mb-4">
               <code>CTRL</code> + <code>T</code>
            </div>
            <div class="mb-4">
               <code>CTRL</code> + <code>W</code>
            </div>
         </div>
      </div>
   </div>
</template>

<script>
export default {
   name: 'WorkspaceTabQueryEmptyState'
};
</script>

<style scoped>
.container {
  padding-top: 15vh;
  opacity: 0.6;
}
</style>
