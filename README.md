# otch
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Уреди App</title>

<style>
body { margin:0; font-family:Arial; background:#f4f4f4; }
.header { background:#1976D2; color:white; padding:15px; text-align:center; }
.container { padding:10px; }

button {
  width:100%; padding:12px; margin:5px 0;
  border:none; background:#1976D2; color:white;
  border-radius:8px; font-size:16px;
}

input {
  width:100%; padding:10px; margin:5px 0;
  border-radius:8px; border:1px solid #ccc;
}

.card {
  background:white; padding:10px; margin:5px 0;
  border-radius:10px;
}

canvas { border:1px solid #000; width:100%; }
#reader { width:100%; }
</style>

<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

</head>

<body>

<div class="header">📱 Уреди система</div>

<div class="container">

<input type="file" id="file">

<button onclick="startScan()">📷 Сканирай баркод</button>

<div id="reader"></div>

<input id="current" placeholder="Текущо показание">
<input id="memory" placeholder="Памет">

<button onclick="saveData()">💾 Запази</button>

<h3>✍️ Подпис</h3>
<canvas id="sig" height="150"></canvas>
<button onclick="saveSignature()">Запази подпис</button>

<h3>📋 Списък</h3>
<div id="list"></div>

<button onclick="exportXLSX()">⬇️ Excel (.xlsx)</button>

</div>

<script>

let devices = JSON.parse(localStorage.getItem("devices") || "[]");
let currentDevice = null;
let signatureImage = "";
let html5QrCode;

/* ---------------- CSV IMPORT ---------------- */
document.getElementById("file").onchange = e => {
    let reader = new FileReader();

    reader.onload = () => {
        devices = reader.result.split("\n").map(l => {
            let c = l.split(",");
            return {
                num: (c[0] || "").trim(), // важно
                addr: c[1],
                ap: c[2],
                current: "",
                memory: "",
                date: "",
                sign: ""
            };
        });

        saveLocal();
        render();
    };

    reader.readAsText(e.target.files[0]);
};

/* ---------------- CAMERA SCAN ---------------- */
function startScan() {

    if (html5QrCode) {
        try { html5QrCode.stop(); html5QrCode.clear(); } catch(e){}
    }

    html5QrCode = new Html5Qrcode("reader");

    Html5Qrcode.getCameras().then(list => {

        if (!list || list.length === 0) {
            alert("Няма камера");
            return;
        }

        const cam = list[0].id;

        html5QrCode.start(
            cam,
            { fps:10, qrbox:250 },
            decodedText => {

                let raw = decodedText;
                let digits = raw.replace(/\D/g, "");
                let code = "";

                // 📌 EAN13 → взима вътрешните 8 цифри
                if (digits.length === 13) {
                    code = digits.substring(3, 11);
                } else {
                    code = digits.slice(1, 9);
                }

                alert(
                    "📷 Сканирано: " + raw +
                    "\n🔢 Цифри: " + digits +
                    "\n➡️ Код: " + code
                );

                currentDevice = devices.find(d =>
                    String(d.num).trim() === code
                );

                setTimeout(() => {
                    alert(
                        currentDevice
                            ? "✔ Намерен уред: " + code
                            : "❌ Не е намерен: " + code
                    );
                }, 200);

                html5QrCode.stop();
            }
        );

    }).catch(err => {
        alert("Camera error: " + err);
    });
}

/* ---------------- SAVE DATA ---------------- */
function saveData() {
    if (!currentDevice) return alert("Сканирай първо");

    currentDevice.current = document.getElementById("current").value;
    currentDevice.memory = document.getElementById("memory").value;
    currentDevice.date = new Date().toLocaleString();
    currentDevice.sign = signatureImage;

    saveLocal();
    render();
}

/* ---------------- SIGNATURE ---------------- */
let canvas = document.getElementById("sig");
let ctx = canvas.getContext("2d");
let drawing = false;

canvas.onmousedown = () => drawing = true;
canvas.onmouseup = () => drawing = false;

canvas.onmousemove = e => {
    if (!drawing) return;
    ctx.lineTo(e.offsetX, e.offsetY);
    ctx.stroke();
};

function saveSignature() {
    signatureImage = canvas.toDataURL("image/png");
    if (currentDevice) currentDevice.sign = signatureImage;
    saveLocal();
}

/* ---------------- RENDER ---------------- */
function render() {
    document.getElementById("list").innerHTML =
        devices.map(d =>
            `<div class="card">
                <b>${d.num}</b><br>
                ${d.addr}<br>
                ${d.current || "-"}
            </div>`
        ).join("");
}

/* ---------------- EXPORT XLSX ---------------- */
function exportXLSX() {

    let data = devices.map(d => ({
        "Уред": d.num,
        "Адрес": d.addr,
        "Апартамент": d.ap,
        "Текущо": d.current,
        "Дата": d.date,
        "Памет": d.memory,
        "Подпис": d.sign
    }));

    let ws = XLSX.utils.json_to_sheet(data);
    let wb = XLSX.utils.book_new();

    XLSX.utils.book_append_sheet(wb, ws, "Уреди");

    XLSX.writeFile(wb, "export.xlsx");
}

/* ---------------- LOCAL ---------------- */
function saveLocal() {
    localStorage.setItem("devices", JSON.stringify(devices));
}

render();

</script>

</body>
</html>
otch
