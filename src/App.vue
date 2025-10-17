<template>
	<div class="container">
		<h1>병원 지원 지도</h1>
		
		<!-- 에러 메시지 표시 -->
		<div v-if="error" class="error-message">
			{{ error }}
		</div>
		
		<!-- 로딩 상태 표시 -->
		<div v-if="loading" class="loading-message">
			데이터를 불러오는 중...
		</div>
		
		<div class="layout">
			<section class="left">
				<h2>병원 등록</h2>
				<HospitalForm v-if="showForm" @added="handleAdded" @close="showForm = false"/>
				<button v-else class="show-btn" @click="showForm = true">
					병원 등록 폼 열기
				</button>
				<!--<div class="list">
					<h3>등록된 병원 ({{ hospitals.length }}개)</h3>
					<ul>
						<li v-for="h in hospitals" :key="h.id">
							<strong>{{ h.HOSPITAL_NM }}</strong>
							<span> — {{ h.ADDR }}</span>
							<span> (모집인원: {{ h.NUM }}명)</span>
						</li>
					</ul>
				</div>
				-->
			</section>
			<section class="right">
				<HospitalMap 
					:hospitals="hospitals" 
					@deleteHospital="handleMapDeleteHospital"
				/>
			</section>
		</div>
	</div>
</template>

<script setup>
import HospitalForm from './components/HospitalForm.vue'
import HospitalMap from './components/HospitalMap.vue'
import { useFirebaseHospitals } from './composables/useFirebaseHospitals.js'
import { ref } from 'vue'
const { hospitals, addHospital, deleteHospital, loading, error } = useFirebaseHospitals()

async function handleAdded(h) {
	try {
		await addHospital(h)
		console.log('병원이 성공적으로 추가되었습니다.')
	} catch (err) {
		console.error('병원 추가 실패:', err)
	}
}
const showForm = ref(true)


async function handleMapDeleteHospital(h) {
	try {
		await deleteHospital(h.id)
		console.log('병원이 성공적으로 삭제되었습니다.')
	} catch (err) {
		console.error('병원 삭제 실패:', err)
	}
}
</script>

<style scoped>
.container {
	max-width: 1920px;
	max-height: 1080px;
	margin: 0 auto;
	padding: 16px;
}

.error-message {
	background: #fee;
	border: 1px solid #fcc;
	color: #c33;
	padding: 12px;
	border-radius: 6px;
	margin-bottom: 16px;
}

.loading-message {
	background: #e3f2fd;
	border: 1px solid #bbdefb;
	color: #1976d2;
	padding: 12px;
	border-radius: 6px;
	margin-bottom: 16px;
	text-align: center;
}

.layout {
	display: grid;
	grid-template-columns: 1fr 2fr;
	gap: 16px;
}
.left {
	background: #f7f7f9;
	border: 1px solid #e6e6ef;
	border-radius: 8px;
	padding: 12px;
	overflow: auto;
}
.right {
	min-height: 560px;
}
.list ul {
	padding-left: 16px;
}
.list li {
	margin: 6px 0;
}
.show-btn {
	position: fixed;
	right: 20px;
	bottom: 20px;
	background-color: #1976d2;
	color: white;
	border: none;
	border-radius: 6px;
	padding: 10px 16px;
	cursor: pointer;
	box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
	z-index: 9999;
}
@media (max-width: 900px) {
	.layout {
		grid-template-columns: 1fr;
	}
	.right {
		min-height: 420px;
	}
}
</style>

