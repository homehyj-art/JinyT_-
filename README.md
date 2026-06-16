<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>세특 자동 생성 시스템 (학교 공유형)</title>

<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

<style>
body{max-width:1100px;margin:auto;padding:20px;font-family:sans-serif}
textarea{width:100%;height:140px;margin-top:10px}
button{margin-top:10px;padding:8px 14px}
select,input{padding:6px;margin-top:10px}
</style>
</head>

<body>

<h2>📘 세특 자동 생성 시스템 (교사용)</h2>

<!-- 로그인 -->
<div>
<input id="teacher" placeholder="교사 이름 입력">
<button onclick="login()">로그인</button>
</div>

<hr>

<!-- 과목 선택 -->
<h3>교과 선택</h3>
<select id="subject">
<option value="국어">국어</option>
<option value="수학">수학</option>
<option value="과학">과학</option>
<option value="사회">사회</option>
<option value="영어">영어</option>
</select>

<!-- 개별 입력 -->
<h3>개별 생성</h3>
<textarea id="input" placeholder="학생 활동"></textarea>
<button onclick="generate()">세특 3버전 생성</button>

<div id="tabs"></div>
<textarea id="output"></textarea>

<hr>

<!-- 반 전체 -->
<h3>반 전체 생성</h3>
<textarea id="bulk" placeholder="학생1 활동\n학생2 활동\n학생3 활동"></textarea>
<button onclick="bulkGenerate()">전체 생성</button>

<hr>

<!-- 엑셀 -->
<h3>엑셀 일괄 생성</h3>
<input type="file" id="excel">
<button onclick="handleExcel()">엑셀 생성</button>

<hr>

<h3>📂 생성 기록</h3>
<div id="history"></div>

<div style="color:red">
※ 개인정보 입력 금지 / 외부 AI 서버 전송됨
</div>

<script>
let versions=[];
let active=0;
let bank=JSON.parse(localStorage.getItem("bank")||"[]");
let history=JSON.parse(localStorage.getItem("history")||"[]");

/* 로그인 */
function login(){
  const name=document.getElementById("teacher").value;
  localStorage.setItem("teacher",name);
  alert(name+" 로그인 완료");
}

/* 교과별 프롬프트 */
function subjectPrompt(text){
  const sub=document.getElementById("subject").value;

  const map={
    국어:"비판적 사고와 해석 중심으로 서술",
    수학:"문제 해결 과정 중심으로 서술",
    과학:"탐구 과정과 실험 중심으로 서술",
    사회:"자료 분석과 비판적 사고 중심",
    영어:"의사소통 능력 중심"
  };

  return text+"\n"+map[sub];
}

/* 개인정보 제거 */
function anonymize(text){
  return text.replace(/[가-힣]{3}/g,"학생");
}

/* 금지어 */
const banned=["수상","대회","논문","특허"];
function filterText(text){
  banned.forEach(w=>text=text.replaceAll(w,""));
  return text;
}

/* 중복검사 */
function isDup(text){
  return bank.some(prev=>{
    const w=text.split(" ");
    for(let i=0;i<w.length-6;i++){
      let chunk=w.slice(i,i+6).join(" ");
      if(prev.includes(chunk)) return true;
    }
  });
}

/* API */
async function ai(prompt){
  const res=await fetch("https://openrouter.ai/api/v1/chat/completions",{
    method:"POST",
    headers:{
      "Authorization":"Bearer YOUR_API_KEY",
      "Content-Type":"application/json"
    },
    body:JSON.stringify({
      model:"mistralai/mistral-7b-instruct",
      messages:[
        {role:"system",content:"세특 작성 교사"},
        {role:"user",content:prompt}
      ]
    })
  });
  const data=await res.json();
  return data.choices?.[0]?.message?.content;
}

/* 생성 */
async function generate(){
  let t=document.getElementById("input").value;

  t=anonymize(t);
  t=filterText(t);
  t=subjectPrompt(t);

  versions=[];

  for(let i=0;i<3;i++){
    let r=await ai(t+" 다른 표현");
    if(!r) continue;
    if(isDup(r)) r=await ai(t+" 완전히 다르게");
    versions.push(r);
  }

  bank.push(versions[0]);
  localStorage.setItem("bank",JSON.stringify(bank));

  history.push({
    teacher:localStorage.getItem("teacher"),
    text:versions[0]
  });
  localStorage.setItem("history",JSON.stringify(history));

  render();
  renderHistory();
}

/* bulk */
async function bulkGenerate(){
  const lines=document.getElementById("bulk").value.split("\n");

  let results=[];

  for(const l of lines){
    let t=subjectPrompt(filterText(anonymize(l)));
    let r=await ai(t);
    results.push(r);
  }

  alert("생성 완료");
  console.log(results);
}

/* 엑셀 */
async function handleExcel(){
  const file=document.getElementById("excel").files[0];
  const reader=new FileReader();

  reader.onload=async(e)=>{
    const data=new Uint8Array(e.target.result);
    const wb=XLSX.read(data,{type:"array"});
    const sheet=wb.Sheets[wb.SheetNames[0]];
    const rows=XLSX.utils.sheet_to_json(sheet);

    let output=[];

    for(const row of rows){
      let t=subjectPrompt(filterText(anonymize(row["활동"]||"")));
      const r=await ai(t);

      output.push({
        이름:row["이름"],
        세특:r
      });
    }

    const ws=XLSX.utils.json_to_sheet(output);
    const newWb=XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(newWb,ws,"결과");
    XLSX.writeFile(newWb,"세특결과.xlsx");
  };

  reader.readAsArrayBuffer(file);
}

/* 출력 */
function render(){
  document.getElementById("output").value=versions[0];

  let t=document.getElementById("tabs");
  t.innerHTML="";

  versions.forEach((v,i)=>{
    let b=document.createElement("button");
    b.innerText="버전"+(i+1);
    b.onclick=()=>{
      document.getElementById("output").value=v;
    };
    t.appendChild(b);
  });
}

/* 기록 */
function renderHistory(){
  let h=document.getElementById("history");
  h.innerHTML="";

  history.slice(-5).forEach(d=>{
    let div=document.createElement("div");
    div.innerText=d.teacher+" : "+d.text.substring(0,50);
    h.appendChild(div);
  });
}

renderHistory();
</script>

</body>
</html>
