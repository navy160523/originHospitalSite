<template>
	<div class="map-container">
		<div class="map-header">
			<h3>병원 위치</h3>
			<p>총 {{ hospitals.length }}개 병원이 등록되어 있습니다</p>
		</div>
		<div ref="mapEl" class="map"></div>
		<div class="hospital-list">
			<h4>병원 목록</h4>
			<h6>서울-나눔터(5명)/서울-누리봄(1명)/서울-새오름터(2명)/서울-양천아이존(3명)/서울-한림대부속한강성심병원(2명)/부산-하모니하우스(1명) 제외</h6>
			<input type="text" placeholder="병원 이름으로 검색(미구현)" />
			<br/><br/>
			<div class="list-items">
				<div 
					v-for="hospital in sortedDocuments" 
					:key="hospital.id"
					class="list-item"
					@click="selectHospital(hospital)"
					:class="{ selected: selectedHospital?.id === hospital.id }"
				>
					<div class="item-info">
						<strong>{{ hospital.HOSPITAL_NM }}</strong>
						<span>{{ hospital.ADDR }}</span>
						<span class="recruitment-count">모집인원: {{ hospital.NUM }}명</span>
						<span class="year-badge">{{ hospital.YEAR }}년</span>
					</div>
					<button @click.stop="deleteHospital(hospital)" class="delete-btn">삭제</button>
				</div>
			</div>
		</div>
	</div>
</template>

<script setup>
import { onMounted, computed, onUnmounted, ref, watch } from 'vue'
import { getFirestore, collection, getDocs } from 'firebase/firestore'
import { db } from '../firebase/config.js' // 'database' 대신 Firestore 'db' 인스턴스 import
import { loadKakao, geocodeAddress } from '../composables/useKakaoLoader.js'
const props = defineProps({
	hospitals: {
		type: Array,
		default: () => []
	}
})
const documents = ref([]);

// 병원 이름 가나다순 정렬된 computed
const sortedDocuments = computed(() => {
  return [...documents.value].sort((a, b) => 
    a.HOSPITAL_NM.localeCompare(b.HOSPITAL_NM, 'ko')
  )
})

const emit = defineEmits(['deleteHospital'])

const mapEl = ref(null)
const selectedHospital = ref(null)
let map = null
let markers = []
let kakaoRef = null

function clearMarkers() {
	markers.forEach((m) => m.setMap(null))
	markers = []
}

function renderMarkers() {
	if (!map || !kakaoRef) return
	clearMarkers()
	
	props.hospitals.forEach((h) => {
		const pos = new kakaoRef.maps.LatLng(h.lat, h.lng)
		const marker = new kakaoRef.maps.Marker({ position: pos, map })
		
		const iwContent = `
			<div style="padding:6px 8px;white-space:nowrap">
				<strong>${h.HOSPITAL_NM}</strong><br/>
				${h.ADDR}<br/>
				모집인원: ${h.NUM}명
			</div>
		`
		const infowindow = new kakaoRef.maps.InfoWindow({ content: iwContent })
		
		kakaoRef.maps.event.addListener(marker, 'click', () => {
			infowindow.open(map, marker)
			selectHospital(h)
		})
		
		markers.push(marker)
	})
	
	if (props.hospitals.length > 0) {
		const bounds = new kakaoRef.maps.LatLngBounds()
		props.hospitals.forEach((h) => bounds.extend(new kakaoRef.maps.LatLng(h.lat, h.lng)))
		map.setBounds(bounds)
	}
}

function selectHospital(hospital) {
	//alert(hospital.HOSPITAL_NM + ' 선택됨'+ hospital.id + "||" + hospital.lat + "," + hospital.lng)
	// selectedHospital.value = selectedHospital.value?.id === hospital.id ? null : hospital
	// if (selectedHospital.value?.id === hospital.id) {
	// 	selectedHospital.value = null
	// } else {
	// 	selectedHospital.value = hospital
		//alert("외부");
		// 지도 확대 및 중심 이동
		if (map && kakaoRef) {
			//alert("내부");
			const latLng = new kakaoRef.maps.LatLng(hospital.lat, hospital.lng)
			map.setLevel(4) // 확대 레벨 (숫자가 작을수록 더 확대됨)
			map.setCenter(latLng)
		}
	// }
}


function deleteHospital(hospital) {
	if (confirm(`${hospital.HOSPITAL_NM}을(를) 삭제하시겠습니까?`)) {
		emit('deleteHospital', hospital)
	}
}


onMounted(async () => {
	 // 1. Kakao SDK 로드
	const { kakao } = await loadKakao()
	kakaoRef = kakao
	// 2. 지도 생성
	map = new kakao.maps.Map(mapEl.value, {
		center: new kakao.maps.LatLng(37.5665, 126.9780), // 초기 위치: 서울
		level: 7
	})

	// 3. Firebase 주소 불러오기
	const querySnapshot = await getDocs(collection(db, 'hospitalData'));

	documents.value = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

	const bounds = new kakao.maps.LatLngBounds()
	const hospitalData = {};

	for (const doc of querySnapshot.docs) {
		const { ADDR, HOSPITAL_NM, NUM, YEAR } = doc.data();
		
		// 주소가 없으면 건너뜁니다.
		if (!ADDR) {
			console.warn('주소가 누락된 데이터가 있습니다.');
			continue;
		}

		// 주소(ADDR)를 키로 사용하여 병원 데이터를 그룹화합니다.
		if (!hospitalData[ADDR]) {
			hospitalData[ADDR] = {
				HOSPITAL_NM: HOSPITAL_NM,
				ADDR: ADDR,
				years: {} // 연도별 모집인원을 저장할 객체
			};
		}
		hospitalData[ADDR].years[YEAR] = NUM;
	}

	// 그룹화된 데이터를 기반으로 마커와 인포윈도우 생성
	for (const addr in hospitalData) {
		const hospital = hospitalData[addr];

		// 주소 변환
		const coords = await geocodeAddress(hospital.ADDR);
		if (!coords) {
			console.warn(`주소 변환 실패: ${hospital.ADDR}`);
			continue;
		}

		const position = new kakao.maps.LatLng(coords.lat, coords.lng);

		// 마커 생성
		const marker = new kakao.maps.Marker({
			map,
			position
		});

		// 인포윈도우 콘텐츠 생성
		let contents = '<div style="width:200px;text-align:center;padding:0px 0;">';
		contents += '<strong>' + hospital.HOSPITAL_NM + '</strong><br/>' + hospital.ADDR;

		// 그룹화된 연도별 데이터를 표시
		for (const year in hospital.years) {
			contents += `<br/>${year} 모집인원: ${hospital.years[year]}명`;
		}
		
		contents += '</div>';

		// 인포윈도우 생성
		const infowindow = new kakao.maps.InfoWindow({
			content: contents
		});

		// 마커에 클릭 이벤트 리스너 등록 (클릭 시 인포윈도우 표시)
		// kakao.maps.event.addListener(marker, 'click', function() {
		// 	infowindow.open(map, marker);
		// });
		let isOpen = false;
		kakao.maps.event.addListener(marker, 'click', function() {
			if (isOpen) {
				infowindow.close();
				isOpen = false;
			} else {
				infowindow.open(map, marker);
				isOpen = true;
			}
		});


		// 인포윈도우를 기본적으로 열기
		//infowindow.open(map, marker);
		
		// 지도 범위 확장
		bounds.extend(position);
	}

	// 7. 마커가 하나 이상일 때 지도를 범위에 맞게 조정
	if (!bounds.isEmpty()) {
		map.setBounds(bounds)
	}
})

onUnmounted(() => {
	clearMarkers()
	map = null
	kakaoRef = null
})

watch(
	() => props.hospitals,
	() => {
		renderMarkers()
	},
	{ deep: true }
)
</script>

<style scoped>
.map-container {
	display: flex;
	flex-direction: column;
	gap: 16px;
	height: 100%;
}

.map-header {
	text-align: center;
	padding: 12px;
	background: #f8f9fa;
	border-radius: 8px;
}

.map-header h3 {
	margin: 0 0 4px 0;
	color: #333;
}

.map-header p {
	margin: 0;
	color: #666;
	font-size: 14px;
}


.map {
	width: 100%;
	height: 400px;
	border: 1px solid #e6e6ef;
	border-radius: 8px;
}

.hospital-list {
	background: #f8f9fa;
	border-radius: 8px;
	padding: 16px;
}

.hospital-list h4 {
	margin: 0 0 12px 0;
	color: #333;
}

.list-items {
	display: flex;
	flex-direction: column;
	gap: 8px;
	max-height: 200px;
	overflow-y: auto;
}

.list-item {
	padding: 12px;
	background: white;
	border-radius: 6px;
	border: 1px solid #e6e6ef;
	cursor: pointer;
	transition: all 0.2s ease;
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.list-item:hover {
	background: #f0f8ff;
	border-color: #3b82f6;
}

.list-item.selected {
	background: #e3f2fd;
	border-color: #1976d2;
}

.item-info {
	display: flex;
	flex-direction: column;
	gap: 4px;
}

.item-info strong {
	color: #333;
	font-size: 14px;
}

.item-info span {
	color: #666;
	font-size: 12px;
}

.recruitment-count {
	color: #1976d2 !important;
	font-weight: 500;
}

.year-badge {
	background: #e3f2fd;
	color: #1976d2;
	padding: 2px 6px;
	border-radius: 4px;
	font-size: 10px;
	font-weight: 500;
}

.delete-btn {
	padding: 4px 8px;
	font-size: 11px;
	background: #ffebee;
	color: #c62828;
	border: 1px solid #ffcdd2;
	border-radius: 4px;
	cursor: pointer;
	transition: all 0.2s ease;
}

.delete-btn:hover {
	background: #ffcdd2;
	border-color: #ef9a9a;
}

@keyframes pulse {
	0% { transform: scale(1); }
	50% { transform: scale(1.1); }
	100% { transform: scale(1); }
}

@media (max-width: 900px) {
	.map {
		height: 250px;
	}
	
	.marker-info {
		min-width: 150px;
		padding: 8px;
	}
	
	.list-items {
		max-height: 150px;
	}
}
</style>

