<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <title>Sousse Smart Watch - Admin Panel 📍</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
    <style>
        /* التنسيق العام */
        body { margin: 0; font-family: 'Segoe UI', Tahoma, sans-serif; }
        #map { height: 100vh; width: 100%; }

        /* القائمة الجانبية */
        .sousse-menu {
            position: fixed; top: 15px; left: 15px; z-index: 1000;
            background: white; padding: 20px; border-radius: 12px;
            box-shadow: 0 5px 20px rgba(0,0,0,0.3); width: 280px; direction: rtl;
        }

        .input-group { margin-top: 10px; display: flex; flex-direction: column; gap: 8px; }
        label { font-size: 12px; font-weight: bold; color: #333; }
        select, input { padding: 10px; border: 1px solid #ccc; border-radius: 6px; font-size: 13px; }

        /* الأزرار */
        .btn-group { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-top: 15px; }
        button { padding: 12px; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; color: white; transition: 0.2s; font-size: 11px; }
        button:active { transform: scale(0.95); }
        
        .btn-water { background: #007bff; }
        .btn-elect { background: #ffc107; color: black; }
        .btn-gas { background: #2d3436; grid-column: span 2; }
        .btn-road { background: #dc3545; grid-column: span 2; }
        .btn-reset { background: #6c757d; grid-column: span 2; }

        /* أيقونات الخريطة */
        .cut-icon-pro {
            display: flex; align-items: center; justify-content: center;
            background: white; border-radius: 50%; box-shadow: 0 0 10px rgba(0,0,0,0.3);
            border: 3px solid; animation: pulse 1.5s infinite; font-size: 20px;
        }

        @keyframes pulse { 0% { transform: scale(1); } 50% { transform: scale(1.1); } 100% { transform: scale(1); } }
        .icon-symbol { position: relative; }
        .block-sign { position: absolute; top: -10px; right: -12px; font-size: 12px; }
        
        .popup-card { text-align: right; direction: rtl; min-width: 170px; }
        .btn-del { background: #ff4757; color: white; border: none; padding: 7px; border-radius: 5px; width: 100%; cursor: pointer; margin-top: 10px; font-weight: bold; }
    </style>
</head>
<body>

<div class="sousse-menu">
    <h3 style="margin:0 0 10px 0; text-align:center;">إدارة خدمات سوسة 📍</h3>
    <div class="input-group">
        <label>🛠️ النوع:</label>
        <select id="reportReason">
            <option value="تعبيد طريق">🏗️ تعبيد طريق</option>
            <option value="صيانة قنوات التطهير">💧 صيانة قنوات التطهير</option>
            <option value="أشغال الكهرباء">⚡ صيانة الكهرباء</option>
            <option value="إصلاح تسرب غاز">🔥 إصلاح تسرب غاز</option>
            <option value="عطب مياه مفاجئ">🚰 عطب مياه مفاجئ</option>
        </select>
        <label>📅 التاريخ والوقت:</label>
        <input type="datetime-local" id="startDate">
        <input type="datetime-local" id="endDate">
    </div>
    <div class="btn-group">
        <button class="btn-water" onclick="setType('water')">💧 قطع الماء</button>
        <button class="btn-elect" onclick="setType('elect')">⚡ قطع الضوء</button>
        <button class="btn-gas" onclick="setType('gas')">🚫 قطع الغاز</button>
        <button class="btn-road" onclick="setType('road')">🚧 طريق مغلق</button>
        <button class="btn-reset" onclick="resetDraft()">❌ إلغاء</button>
    </div>
    <div id="status" style="font-size:11px; color:red; text-align:center; margin-top:10px; font-weight:bold;">جاهز للعمل</div>
</div>

<div id="map"></div>

<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onValue, remove } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

    // إعدادات Firebase
    const firebaseConfig = {
        apiKey: "AIzaSyDbc9XVF3W7C-DjfC7-TgMLPP8b4Hixqf0",
        databaseURL: "https://sousse-watch-default-rtdb.europe-west1.firebasedatabase.app",
    };

    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);
    const reportsRef = ref(db, 'reports');

    // إعداد الخريطة
    var map = L.map('map', { doubleClickZoom: false }).setView([35.8256, 10.6084], 14);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

    var currentType = null, points = [], liveMarkers = L.layerGroup().addTo(map);
    var liveLine = L.polyline([], {color: '#666', weight: 2, dashArray: '5,5'}).addTo(map);

    // وظائف التحكم
    window.setType = (type) => { 
        currentType = type; points = []; liveMarkers.clearLayers(); liveLine.setLatLngs([]);
        document.getElementById('status').innerText = (type === 'road') ? "📍 ارسم المسار وانقر مرتين للحفظ" : "📍 حدد 4 نقاط للمنطقة";
    };

    window.resetDraft = () => { currentType = null; points = []; liveMarkers.clearLayers(); liveLine.setLatLngs([]); document.getElementById('status').innerText = "جاهز"; };

    map.on('click', (e) => {
        if (!currentType) return;
        points.push(e.latlng);
        L.circleMarker(e.latlng, {radius: 5, color: '#000', fillColor: '#fff', fillOpacity: 1}).addTo(liveMarkers);
        liveLine.setLatLngs(points);
        if ((currentType === 'water' || currentType === 'elect' || currentType === 'gas') && points.length === 4) saveToFirebase();
    });

    map.on('dblclick', () => { if (currentType === 'road' && points.length > 1) saveToFirebase(); });

    function saveToFirebase() {
        const start = document.getElementById('startDate').value;
        const end = document.getElementById('endDate').value;
        if(!start) { alert("أدخل وقت البداية!"); return; }

        push(reportsRef, {
            type: currentType,
            coords: points,
            reason: document.getElementById('reportReason').value,
            start: start.replace('T',' '),
            end: end ? end.replace('T',' ') : "غير محدد"
        });
        resetDraft();
    }

    // جلب البيانات وعرضها
    onValue(reportsRef, (snapshot) => {
        map.eachLayer(l => { if((l instanceof L.Polygon || l instanceof L.Polyline || l instanceof L.Marker) && l !== liveLine) map.removeLayer(l); });
        snapshot.forEach((child) => {
            const v = child.val();
            const id = child.key;
            
            let color = v.type === 'road' ? '#dc3545' : (v.type === 'water' ? '#007bff' : (v.type === 'elect' ? '#ffc107' : '#2d3436'));
            
            let layer;
            if(v.type === 'road') {
                layer = L.polyline(v.coords, {color: color, weight: 7, opacity: 0.8}).addTo(map);
            } else {
                layer = L.polygon(v.coords, {color: color, fillOpacity: 0.3, weight: 2}).addTo(map);
                const center = layer.getBounds().getCenter();
                let iconChar = v.type === 'water' ? '💧' : (v.type === 'elect' ? '⚡' : '🔥');
                
                L.marker(center, {
                    icon: L.divIcon({ 
                        className: 'cut-icon-pro', 
                        html: `<div class="icon-symbol" style="color:${color}">${iconChar}<span class="block-sign">🚫</span></div>`, 
                        iconSize: [40, 40], iconAnchor: [20, 20]
                    })
                }).addTo(map);
            }

            layer.bindPopup(`
                <div class="popup-card">
                    <b>${v.type === 'road' ? '🚧 طريق مغلق' : '🚫 انقطاع خدمة'}</b><br>
                    🛠️ <b>السبب:</b> ${v.reason}<br>
                    📅 <b>من:</b> ${v.start}<br>
                    ⏳ <b>إلى:</b> ${v.end}<br>
                    <button class="btn-del" onclick="deleteReport('${id}')">🗑️ حذف</button>
                </div>
            `);
        });
    });

    window.deleteReport = (id) => { if(confirm("حذف الإشعار؟")) remove(ref(db, `reports/${id}`)); };
</script>
</body>
</html>
