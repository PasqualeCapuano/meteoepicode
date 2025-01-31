<template>
	<v-app-bar elevation="2" class="gradient px-4" height="120">
		<v-app-bar-title class="text-center w-100">
			<span class="text-h2 font-weight-bold">Meteo Epicode</span>
		</v-app-bar-title>
	</v-app-bar>
</template>

<script>
export default {
	name: "Banner",
};
</script>

<style scoped>
.gradient {
	background: linear-gradient(45deg, #9c27b0, #e1bee7) !important;
	color: white !important;
}

.w-100 {
	width: 100%;
}

.text-center {
	text-align: center;
}
</style>
