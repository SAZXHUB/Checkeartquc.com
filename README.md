<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ระบบตรวจจับแผ่นดินไหว</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
            text-align: center;
        }
        #earthquakeInfo {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #ddd;
            background-color: #f9f9f9;
        }
        #riskAssessment {
            margin-top: 10px;
            font-weight: bold;
        }
        #aftershockPrediction {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #ffc107;
            background-color: #fff8e1;
        }
        .high-risk {
            color: #d32f2f;
        }
        .medium-risk {
            color: #f57c00;
        }
        .low-risk {
            color: #388e3c;
        }
        .arrival-time {
            font-size: 1.2em;
            margin-top: 10px;
            font-weight: bold;
        }
        .countdown {
            font-size: 1.5em;
            font-weight: bold;
            color: #d32f2f;
            background-color: #ffebee;
            padding: 10px;
            border-radius: 5px;
            margin: 10px 0;
            animation: pulse 1s infinite;
        }
        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.8; }
            100% { opacity: 1; }
        }
        .countdown-container {
            border: 2px solid #d32f2f;
            margin-top: 15px;
            padding: 10px;
            border-radius: 5px;
            background-color: #fff8e1;
        }
        .wave-type {
            font-weight: bold;
            margin-right: 5px;
        }
    </style>
</head>
<body>
    <h1>ระบบติดตามแผ่นดินไหว</h1>
    <div id="locationInfo">กำลังค้นหาตำแหน่ง...</div>
    <div id="earthquakeInfo">รอการโหลดข้อมูล...</div>
    <div id="riskAssessment"></div>
    <div id="aftershockPrediction"></div>
    <div id="waveArrival"></div>
    <div id="countdownDisplay"></div>

    <script>
        // ตัวแปรสำหรับการจัดการนับถอยหลัง
        let countdownInterval;
        let pWaveArrivalTime = null;
        let sWaveArrivalTime = null;

        // ฟังก์ชันสำหรับรับตำแหน่งปัจจุบัน
        function getCurrentLocation() {
            return new Promise((resolve, reject) => {
                if ("geolocation" in navigator) {
                    navigator.geolocation.getCurrentPosition(
                        (position) => {
                            resolve({
                                latitude: position.coords.latitude,
                                longitude: position.coords.longitude
                            });
                        },
                        (error) => {
                            reject(error);
                        }
                    );
                } else {
                    reject(new Error("Geolocation is not supported by this browser."));
                }
            });
        }

        // ฟังก์ชันดึงข้อมูลแผ่นดินไหวจาก USGS API
        async function fetchEarthquakeData(lat, lon) {
            try {
                // ใช้ USGS API ซึ่งเป็น API ฟรีสำหรับข้อมูลแผ่นดินไหว
                const response = await fetch(
                    `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_day.geojson`
                );
                const data = await response.json();

                // กรองแผ่นดินไหวภายใน 5000 กม.
                const nearbyEarthquakes = data.features.filter(quake => {
                    const [qLon, qLat] = quake.geometry.coordinates;
                    const distance = calculateDistance(lat, lon, qLat, qLon);
                    return distance <= 5000; // 5000 กม.
                });

                return nearbyEarthquakes;
            } catch (error) {
                console.error("เกิดข้อผิดพลาดในการดึงข้อมูล:", error);
                return [];
            }
        }

        // ฟังก์ชันคำนวณระยะทาง (Haversine formula)
        function calculateDistance(lat1, lon1, lat2, lon2) {
            const R = 6371; // รัศมีโลกในหน่วยกิโลเมตร
            const dLat = (lat2 - lat1) * Math.PI / 180;
            const dLon = (lon2 - lon1) * Math.PI / 180;
            const a = 
                Math.sin(dLat/2) * Math.sin(dLat/2) +
                Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * 
                Math.sin(dLon/2) * Math.sin(dLon/2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
            return R * c;
        }

        // ฟังก์ชันประเมินความเสี่ยง
        function assessEarthquakeRisk(earthquakes) {
            if (earthquakes.length === 0) return "ความเสี่ยงต่ำ - ไม่พบแผ่นดินไหวใกล้เคียง";

            let highestMagnitude = 0;
            let closestEarthquake = null;

            earthquakes.forEach(quake => {
                const magnitude = quake.properties.mag;
                if (magnitude > highestMagnitude) {
                    highestMagnitude = magnitude;
                    closestEarthquake = quake;
                }
            });

            let riskLevel = "ความเสี่ยงต่ำ";
            let riskClass = "low-risk";
            
            if (highestMagnitude >= 7) {
                riskLevel = "ความเสี่ยงสูงมาก";
                riskClass = "high-risk";
            } else if (highestMagnitude >= 5) {
                riskLevel = "ความเสี่ยงสูง";
                riskClass = "high-risk";
            } else if (highestMagnitude >= 4) {
                riskLevel = "ความเสี่ยงปานกลาง";
                riskClass = "medium-risk";
            } else if (highestMagnitude >= 3) {
                riskLevel = "ความเสี่ยงต่ำ";
                riskClass = "low-risk";
            }

            return {
                text: `${riskLevel} - แผ่นดินไหวขนาด ${highestMagnitude.toFixed(1)} ที่ตรวจพบ`,
                class: riskClass,
                magnitude: highestMagnitude
            };
        }

        // ฟังก์ชันคาดเดา Aftershock
        function predictAftershock(mainEarthquake, userLocation) {
            if (!mainEarthquake) return { probability: 0, magnitude: 0 };

            const [qLon, qLat, depth] = mainEarthquake.geometry.coordinates;
            const mainMagnitude = mainEarthquake.properties.mag;
            const distance = calculateDistance(userLocation.latitude, userLocation.longitude, qLat, qLon);
            const timeSinceQuake = (Date.now() - mainEarthquake.properties.time) / (1000 * 60 * 60); // ชั่วโมง

            // สูตร Omori Law สำหรับการคาดเดา aftershock
            // โอกาสของการเกิด aftershock ลดลงตามเวลา
            const k = 0.9; // ค่าคงที่
            const c = 0.05; // ค่าคงที่
            const p = 1.2; // ค่าคงที่ (ปกติอยู่ระหว่าง 0.7-1.5)

            // คำนวณความน่าจะเป็นของ aftershock
            let aftershockProbability = 0;
            
            if (mainMagnitude >= 5) {
                aftershockProbability = k / Math.pow(timeSinceQuake + c, p);
                // ปรับตามระยะทางจากจุดศูนย์กลาง
                aftershockProbability *= Math.exp(-distance / 500);
                // ปรับตามขนาดของแผ่นดินไหวหลัก
                aftershockProbability *= (mainMagnitude - 4) / 5;
                
                // จำกัดค่าความน่าจะเป็นระหว่าง 0-1
                aftershockProbability = Math.min(Math.max(aftershockProbability, 0), 0.95);
            }

            // คาดเดาขนาดของ aftershock
            // โดยทั่วไป aftershock จะมีขนาดเล็กกว่าแผ่นดินไหวหลัก
            const aftershockMagnitude = mainMagnitude - 1.2 - (timeSinceQuake / 24) * 0.1;

            // คำนวณเวลาที่คลื่นแผ่นดินไหวจะเดินทางมาถึง
            const waveArrivalTime = calculateWaveArrivalTime(distance, depth);

            return {
                probability: aftershockProbability * 100, // เปลี่ยนเป็นเปอร์เซ็นต์
                magnitude: Math.max(aftershockMagnitude, 2.0), // กำหนดค่าต่ำสุด
                arrivalTime: waveArrivalTime,
                distance: distance
            };
        }

        // ฟังก์ชันคำนวณเวลาที่คลื่นแผ่นดินไหวจะมาถึง
        function calculateWaveArrivalTime(distance, depth) {
            // ความเร็วของคลื่น P-wave ประมาณ 6-8 กม./วินาที
            const pWaveVelocity = 7; // กม./วินาที
            
            // ความเร็วของคลื่น S-wave ประมาณ 3-4 กม./วินาที
            const sWaveVelocity = 3.5; // กม./วินาที
            
            // คำนวณระยะทางจริงโดยใช้ทฤษฎีปีทาโกรัส (ความลึก + ระยะทางผิวโลก)
            const actualDistance = Math.sqrt(Math.pow(distance, 2) + Math.pow(depth, 2));
            
            // คำนวณเวลาที่คลื่น P และ S จะมาถึง (ในหน่วยวินาที)
            const pWaveTime = actualDistance / pWaveVelocity;
            const sWaveTime = actualDistance / sWaveVelocity;
            
            return {
                pWave: pWaveTime, // วินาที
                sWave: sWaveTime  // วินาที
            };
        }

        // ฟังก์ชันแสดงผลการคาดเดา Aftershock
        function displayAftershockPrediction(prediction, mainMagnitude) {
            const aftershockEl = document.getElementById('aftershockPrediction');
            const waveArrivalEl = document.getElementById('waveArrival');
            
            if (prediction.probability < 5) {
                aftershockEl.innerHTML = `<span class="low-risk">โอกาสเกิด Aftershock ต่ำมาก (${prediction.probability.toFixed(1)}%)</span>`;
                waveArrivalEl.innerHTML = '';
                stopCountdown(); // หยุดการนับถอยหลัง
                return;
            }
            
            let riskClass = "low-risk";
            if (prediction.probability > 70) riskClass = "high-risk";
            else if (prediction.probability > 30) riskClass = "medium-risk";
            
            const now = Date.now();
            pWaveArrivalTime = now + prediction.arrivalTime.pWave * 1000;
            sWaveArrivalTime = now + prediction.arrivalTime.sWave * 1000;
            
            aftershockEl.innerHTML = `
                <h3>การคาดเดา Aftershock</h3>
                <p>โอกาสเกิด: <span class="${riskClass}">${prediction.probability.toFixed(1)}%</span></p>
                <p>ขนาดที่คาดเดา: ${prediction.magnitude.toFixed(1)}</p>
                <p>ระยะทาง: ${prediction.distance.toFixed(1)} กม.</p>
            `;
            
            // แสดงเฉพาะเมื่อแผ่นดินไหวมีขนาดใหญ่พอและยังไม่ถึงเวลาที่คลื่นจะมาถึง
            if (mainMagnitude >= 5 && prediction.arrivalTime.sWave > 0) {
                const pMinutesLeft = Math.floor(prediction.arrivalTime.pWave / 60);
                const pSecondsLeft = Math.floor(prediction.arrivalTime.pWave % 60);
                
                const sMinutesLeft = Math.floor(prediction.arrivalTime.sWave / 60);
                const sSecondsLeft = Math.floor(prediction.arrivalTime.sWave % 60);
                
                waveArrivalEl.innerHTML = `
                    <h3>การคาดเดาเวลาที่คลื่นแผ่นดินไหวจะมาถึง</h3>
                    <p class="arrival-time">คลื่น P-wave: ${pMinutesLeft} นาที ${pSecondsLeft} วินาที</p>
                    <p class="arrival-time">คลื่น S-wave: ${sMinutesLeft} นาที ${sSecondsLeft} วินาที</p>
                    <p><small>* P-wave คือคลื่นแรงอัด (มาถึงก่อน, ความรุนแรงน้อยกว่า)<br>
                    * S-wave คือคลื่นแรงเฉือน (มาถึงหลัง, ทำให้เกิดการสั่นไหวรุนแรง)</small></p>
                `;
                
                // เริ่มนับถอยหลัง
                startCountdown(prediction.arrivalTime.pWave, prediction.arrivalTime.sWave);
            } else {
                waveArrivalEl.innerHTML = '';
                stopCountdown(); // หยุดการนับถอยหลัง
            }
        }

        // ฟังก์ชันเริ่มนับถอยหลัง
        function startCountdown(pWaveSeconds, sWaveSeconds) {
            // ล้างการนับถอยหลังเดิม (ถ้ามี)
            stopCountdown();
            
            // ตั้งค่าเวลาที่คลื่นจะมาถึง
            const now = Date.now();
            pWaveArrivalTime = now + pWaveSeconds * 1000;
            sWaveArrivalTime = now + sWaveSeconds * 1000;
            
            // เริ่มนับถอยหลังใหม่
            countdownInterval = setInterval(updateCountdown, 1000);
            
            // อัพเดทการนับถอยหลังทันที
            updateCountdown();
        }

        // ฟังก์ชันอัพเดทการนับถอยหลัง
        function updateCountdown() {
            const now = Date.now();
            const countdownEl = document.getElementById('countdownDisplay');
            
            let pWaveTimeLeft = Math.max(0, Math.floor((pWaveArrivalTime - now) / 1000));
            let sWaveTimeLeft = Math.max(0, Math.floor((sWaveArrivalTime - now) / 1000));
            
            // ตรวจสอบว่าคลื่นมาถึงแล้วหรือยัง
            const pWaveArrived = pWaveTimeLeft <= 0;
            const sWaveArrived = sWaveTimeLeft <= 0;
            
            if (pWaveArrived && sWaveArrived) {
                countdownEl.innerHTML = '<div class="countdown-container">' +
                    '<h3>สถานะคลื่นแผ่นดินไหว</h3>' +
                    '<p><span class="wave-type">P-wave:</span> <span class="high-risk">มาถึงแล้ว</span></p>' +
                    '<p><span class="wave-type">S-wave:</span> <span class="high-risk">มาถึงแล้ว</span></p>' +
                    '<p class="countdown">คลื่นแผ่นดินไหวมาถึงแล้ว!</p>' +
                    '</div>';
                
                // ยกเลิกการนับถอยหลังหลังจาก 5 นาที (300 วินาที)
                if (sWaveTimeLeft < -300) {
                    stopCountdown();
                    countdownEl.innerHTML = '';
                }
            } else {
                const pMinutes = Math.floor(pWaveTimeLeft / 60);
                const pSeconds = pWaveTimeLeft % 60;
                
                const sMinutes = Math.floor(sWaveTimeLeft / 60);
                const sSeconds = sWaveTimeLeft % 60;
                
                let pWaveDisplay = pWaveArrived ?
                    '<span class="high-risk">มาถึงแล้ว</span>' :
                    `<span class="countdown">${pMinutes}:${pSeconds.toString().padStart(2, '0')}</span>`;
                
                let sWaveDisplay = sWaveArrived ?
                    '<span class="high-risk">มาถึงแล้ว</span>' :
                    `<span class="countdown">${sMinutes}:${sSeconds.toString().padStart(2, '0')}</span>`;
                
                countdownEl.innerHTML = '<div class="countdown-container">' +
                    '<h3>นับถอยหลังคลื่นแผ่นดินไหว</h3>' +
                    '<p><span class="wave-type">P-wave:</span> ' + pWaveDisplay + '</p>' +
                    '<p><span class="wave-type">S-wave:</span> ' + sWaveDisplay + '</p>' +
                    (pWaveArrived && !sWaveArrived ? 
                        '<p class="countdown">เตรียมพร้อมรับมือคลื่น S!</p>' : 
                        '') +
                    '</div>';
            }
        }
        
        // ฟังก์ชันหยุดนับถอยหลัง
        function stopCountdown() {
            if (countdownInterval) {
                clearInterval(countdownInterval);
                countdownInterval = null;
            }
        }

        // ฟังก์ชันหลักที่รวบรวมและแสดงผล
        async function updateEarthquakeInfo() {
            try {
                // รับตำแหน่งปัจจุบัน
                const location = await getCurrentLocation();
                
                // แสดงตำแหน่ง
                document.getElementById('locationInfo').innerHTML = 
                    `ตำแหน่งปัจจุบัน: ละติจูด ${location.latitude.toFixed(4)}, ลองจิจูด ${location.longitude.toFixed(4)}`;

                // ดึงข้อมูลแผ่นดินไหว
                const earthquakes = await fetchEarthquakeData(
                    location.latitude, 
                    location.longitude
                );

                // เรียงลำดับแผ่นดินไหวตามขนาด (จากมากไปน้อย)
                earthquakes.sort((a, b) => b.properties.mag - a.properties.mag);

                // แสดงรายละเอียดแผ่นดินไหว
                const earthquakeInfoEl = document.getElementById('earthquakeInfo');
                if (earthquakes.length === 0) {
                    earthquakeInfoEl.innerHTML = "ไม่พบแผ่นดินไหวใกล้เคียง";
                    stopCountdown(); // หยุดการนับถอยหลัง
                } else {
                    const quakeDetails = earthquakes.slice(0, 5).map(quake => {
                        const [lon, lat, depth] = quake.geometry.coordinates;
                        const distance = calculateDistance(
                            location.latitude, 
                            location.longitude, 
                            lat, 
                            lon
                        );
                        const timeDiff = Math.floor((Date.now() - quake.properties.time) / (1000 * 60)); // นาที
                        let timeAgo = `${timeDiff} นาทีที่แล้ว`;
                        if (timeDiff > 60) {
                            timeAgo = `${Math.floor(timeDiff / 60)} ชั่วโมง ${timeDiff % 60} นาทีที่แล้ว`;
                        }
                        
                        return `
                            <div style="text-align: left; margin-bottom: 10px; padding-bottom: 5px; border-bottom: 1px dotted #ccc;">
                                <b>ขนาด ${quake.properties.mag.toFixed(1)}</b> 
                                • ระยะทาง ${distance.toFixed(0)} กม. 
                                • ความลึก ${depth.toFixed(1)} กม.
                                <br>• สถานที่: ${quake.properties.place || 'ไม่ระบุ'}
                                <br>• เวลา: ${new Date(quake.properties.time).toLocaleString()} (${timeAgo})
                            </div>
                        `;
                    }).join('');
                    
                    earthquakeInfoEl.innerHTML = `
                        <h3>พบ ${earthquakes.length} แผ่นดินไหวใกล้เคียง (แสดง 5 อันดับแรก)</h3>
                        ${quakeDetails}
                    `;
                }

                // ประเมินความเสี่ยง
                const riskAssessment = assessEarthquakeRisk(earthquakes);
                document.getElementById('riskAssessment').innerHTML = 
                    `<span class="${riskAssessment.class}">${riskAssessment.text}</span>`;

                // คาดเดา Aftershock และเวลาที่คลื่นจะมาถึงจากแผ่นดินไหวที่ใหญ่ที่สุด
                if (earthquakes.length > 0) {
                    const mainEarthquake = earthquakes[0]; // แผ่นดินไหวที่มีขนาดใหญ่ที่สุด
                    const aftershockPrediction = predictAftershock(mainEarthquake, location);
                    displayAftershockPrediction(aftershockPrediction, mainEarthquake.properties.mag);
                } else {
                    document.getElementById('aftershockPrediction').innerHTML = '';
                    document.getElementById('waveArrival').innerHTML = '';
                    document.getElementById('countdownDisplay').innerHTML = '';
                    stopCountdown(); // หยุดการนับถอยหลัง
                }

            } catch (error) {
                document.getElementById('locationInfo').innerHTML = 
                    `เกิดข้อผิดพลาด: ${error.message}`;
            }
        }

        // อัปเดตข้อมูลทุก 1 นาที
        updateEarthquakeInfo();
        setInterval(updateEarthquakeInfo, 600);
    </script>
</body>
</html>
