---
title: "Tax code calculator according to the Italian specification"
---

This italian tax code calculator is totally offline and does not share your data with any server!

<ul id="saved-taxcodes" style="display: none;"> 
  Saved tax codes <button id="export-taxcode-button" class="text-button">(export)</button>:
  
  <div id="saved-taxcodes-list"></div>
 
  <dialog id="export-taxcode-dialog" style="max-width: 400px; padding: 1em;">
    <button id="export-taxcode-close" class="text-button" style="float: right;">Close</button>
    <h3>Scan the QR code or copy the link</h3>
    <div id="export-taxcode-qr"></div>
    </br>
    <button id="export-taxcode-link" class="text-button" style="white-space: normal; word-break: break-all; text-align: left;"></button>
  </dialog>
</ul>

<dialog id="import-taxcode-dialog" style="max-width: 400px; padding: 1em;">
  <strong 
    style="display: block; text-align: center; white-space: normal; word-wrap: break-word;">
    You are importing tax codes from another device, those currently saved (if any) on this device will be OVERWRITTEN
  </strong>
  <br>
  
  <strong 
    style="display: block; text-align: center; white-space: normal; word-wrap: break-word;">
    Do you really want to proceed?
  </strong>
  <br>
  
  <button id="import-taxcode-no" class="text-button"
    style="display: block; margin: 0.5em auto;">
    No
  </button>
  <button id="import-taxcode-yes" class="text-button"
    style="display: block; margin: 0.5em auto; white-space: normal; word-break: break-word;">
    Yes, please overwrite my tax codes with the ones to be imported
  </button>
</dialog>

<form id="form-cf"">
  <label for="firstName">First name:</label>
  <input type="text" id="firstName" required value="Sofia">

  <label for="lastName">Last name:</label>
  <input type="text" id="lastName" required value="Rossi">

  <label for="birthDate">Date of birth:</label>
  <input type="date" id="birthDate" required value="1990-01-01">

  <label for="sesso">Sex:</label>
  <input class="button" type="radio" name="sex" id="female" checked/>
  <label for="female">F</label>
  <input class="button" type="radio" name="sex" id="male" />
  <label for="male">M</label> 
  
  <label id="municipalityLabel" for="municipalityCode">Municipality:</label>
  <input type="text" id="municipalityCode" required value="Abano Terme (A001)">
  <ul id="municipalitySuggestions" style="list-style-type: none; padding: 0; margin: 0; display: none;"></ul>

</form>

<h3>Tax code:</h3>
<p id="cf-result"></p>

<script src="https://cdn.jsdelivr.net/npm/qrcodejs/qrcode.min.js"></script>
<script>
  const urlParams = new URLSearchParams(window.location.search);
  const export_taxcode_dialog = document.getElementById('export-taxcode-dialog');
  const export_taxcode_button = document.getElementById('export-taxcode-button');
  const export_taxcode_close = document.getElementById('export-taxcode-close');
  const export_taxcode_qr = document.getElementById('export-taxcode-qr');
  const export_taxcode_link = document.getElementById('export-taxcode-link');
  const import_taxcode_dialog = document.getElementById('import-taxcode-dialog');
  const import_taxcode_yes = document.getElementById('import-taxcode-yes');
  const import_taxcode_no = document.getElementById('import-taxcode-no');
    
  let selectedMunicipalityCode = "A001";

  let codiciCatastali = [];

  async function loadMunicipalityData() {
    try {
      const response = await fetch("/cf/ANPR_archivio_comuni.csv");
      const csvText = await response.text();
      
      const rows = csvText.split('\n').slice(1);
      codiciCatastali = codiciCatastali.concat(rows.map(row => {
        const columns = row.split(',');
        
        if (columns.length != 18) {
          return null;
        }
        
        const codiceCatastale = columns[4].trim().replace(/^"+|"+$/g, '');
        const name = columns[5].trim().replace(/^"+|"+$/g, '');
        const old = columns[2].trim().replace(/^"+|"+$/g, '') === "9999-12-31" ? false : true;
        
        if (!codiceCatastale || !name) {
          return null;
        }
      
        return {
          codiceCatastale: codiceCatastale.toUpperCase(),
          name: name.toUpperCase(),
          old: old,
          country: false
        };
      }).filter(item => item !== null));
    } catch (error) {
      console.error("Error loadMunicipalityData:", error);
    }
  }
  
  async function loadCountryData() {
    try {
      const response = await fetch("/cf/istat_stati_esteri.csv");
      const csvText = await response.text();
      
      const rows = csvText.split('\n').slice(1);
      codiciCatastali = codiciCatastali.concat(rows.map(row => {
        const columns = row.split(';');
        
        if (columns.length != 15) {
          return null;
        }
        
        const codiceCatastale = columns[9].trim();
        const name = columns[6].trim();
        
        if (!codiceCatastale || !name || codiceCatastale == "n.d." || codiceCatastale == "n.d") {
          return null;
        }
      
        return {
          codiceCatastale: codiceCatastale.toUpperCase(),
          name: name.toUpperCase(),
          old: false,
          country: true
        };
      }).filter(item => item !== null));
    } catch (error) {
      console.error("Error loadCountryData:", error);
    }
  }
  
  async function loadCountryDataOld() {
    try {
      const response = await fetch("/cf/istat_stati_esteri_old.csv");
      const csvText = await response.text();
      
      const rows = csvText.split('\n').slice(1);
      codiciCatastali = codiciCatastali.concat(rows.map(row => {
        const columns = row.split(';');
        
        if (columns.length != 10) {
          return null;
        }
        
        const codiceCatastale = columns[4].trim();
        const name = columns[7].trim();
        
        if (!codiceCatastale || !name || codiceCatastale == "n.d." || codiceCatastale == "n.d") {
          return null;
        }
      
        return {
          codiceCatastale: codiceCatastale.toUpperCase(),
          name: name.toUpperCase(),
          old: true,
          country: true
        };
      }).filter(item => item !== null));
    } catch (error) {
      console.error("Error loadCountryDataOld:", error);
    }
  }
  
  async function loadCodiciCatastali() {
    await loadMunicipalityData();
    await loadCountryData();
    await loadCountryDataOld();
    
    codiciCatastali = [...new Map(codiciCatastali
      .map(obj => [`${obj.codiceCatastale}-${obj.name}`, obj])
    ).values()];
  }

  function findMunicipalityCode(name) {
    return codiciCatastali.filter(item => item.name.toLowerCase().includes(name.toLowerCase())).slice(0, 10);
  }

  function updateSuggestions(matches) {
    const suggestionsList = document.getElementById("municipalitySuggestions");
    suggestionsList.innerHTML = "";
    if (matches.length > 0) {
      matches.forEach(item => {
        const li = document.createElement("li");
        
        let name = item.name;
        
        if (item.country && item.old) {
          name = name + " - Supressed foreign state";
        } else if (item.country) {
          name = name + " - Foreign state";
        } else if (item.old) {
          name = name + " - Supressed municipality";
        }
        
        li.textContent = `${name} (${item.codiceCatastale})`;
        li.addEventListener("click", () => selectMunicipality(item));
        suggestionsList.appendChild(li);
      });
      suggestionsList.style.display = "block";
    } else {
      suggestionsList.style.display = "none";
    }
  }

  function selectMunicipality(item) {
    document.getElementById("municipalityCode").value = `${item.name} (${item.codiceCatastale})`;
    document.getElementById("municipalitySuggestions").style.display = "none";
    selectedMunicipalityCode = item.codiceCatastale;
    calculateTaxCode();
  }

  function calculateTaxCode() {
    const firstName = document.getElementById("firstName").value.trimStart().trimEnd();
    const lastName = document.getElementById("lastName").value.trimStart().trimEnd();
    const birthDate = new Date(document.getElementById("birthDate").value);
    const sex = document.querySelector('input[name="sex"]:checked').id;
    let municipalityCode = document.getElementById("municipalityCode").value;
    
    if (selectedMunicipalityCode) {
      municipalityCode = selectedMunicipalityCode;
    }
      
    function extractConsonants(str) {
      return str.replace(/[aeiou]/gi, '').toUpperCase();
    }

    function extractVowels(str) {
      return str.replace(/[^aeiou]/gi, '').toUpperCase();
    }
    
    function calculateLastName(lastName) {
      let lastNameCode;
      const lastNameConsonants = extractConsonants(lastName).slice(0, 3);
      const lastNameVowels = extractVowels(lastName).slice(0, 3);
      
      lastNameCode = lastNameConsonants;
      
      if (lastNameConsonants.length < 3) {
        lastNameCode = lastNameCode + lastNameVowels;
      }
      
      lastNameCode = lastNameCode + "XXX";
      
      return lastNameCode.slice(0, 3);
    }
    
    function calculateFirstName(firstName) {
      let firstNameCode;
      const firstNameConsonants = extractConsonants(firstName).slice(0, 4);
      const firstNameVowels = extractVowels(firstName).slice(0, 3);
      
      if (firstNameConsonants.length > 3) {
        firstNameCode = firstNameConsonants[0] + firstNameConsonants[2] + firstNameConsonants[3];
      } else {
        firstNameCode = firstNameConsonants;
      }
      
      if (firstNameConsonants.length < 3) {
        firstNameCode = firstNameCode + firstNameVowels;
      }
      
      firstNameCode = firstNameCode + "XXX";
      
      return firstNameCode.slice(0, 3);
    }

    function calculateBirthDate(date, sex) {
      const year = date.getFullYear().toString().slice(2);
      const month = "ABCDEHLMPRST"[(date.getMonth())];
      const daysex = sex === 'male' ? date.getDate() : date.getDate() + 40;

      return year + month + (daysex < 10 ? "0" + daysex : daysex);
    }
    
    function calculateControlCharacter(str) {
      const oddValues = { 0: 1, 1: 0, 2: 5, 3: 7, 4: 9, 5: 13, 6: 15, 7: 17, 8: 19, 9: 21, 
        A: 1, B: 0, C: 5, D: 7, E: 9, F: 13, G: 15, H: 17, I: 19, J: 21, K: 2, L: 4, M: 18, N: 20, 
        O: 11, P: 3, Q: 6, R: 8, S: 12, T: 14, U: 16, V: 10, W: 22, X: 25, Y: 24, Z: 23 };
      const evenValues = { 0: 0, 1: 1, 2: 2, 3: 3, 4: 4, 5: 5, 6: 6, 7: 7, 8: 8, 9: 9, 
        A: 0, B: 1, C: 2, D: 3, E: 4, F: 5, G: 6, H: 7, I: 8, J: 9, K: 10, L: 11, M: 12, N: 13, 
        O: 14, P: 15, Q: 16, R: 17, S: 18, T: 19, U: 20, V: 21, W: 22, X: 23, Y: 24, Z: 25 };
      
      let val = 0
      for (let i = 0; i < 15; i = i + 1) {
        const c = str[i]
        val += i % 2 !== 0 ? evenValues[c] : oddValues[c]
      }
      val = val % 26
      
      return 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.charAt(val)
    }

    const lastNameCode = calculateLastName(lastName);
    const firstNameCode = calculateFirstName(firstName);
    const birthData = calculateBirthDate(birthDate, sex);
    
    let taxCode = (lastNameCode + firstNameCode + birthData + municipalityCode).toUpperCase();
    taxCode = taxCode + calculateControlCharacter(taxCode);
    
    let savedTaxCodes = getSavedTaxCodes();
    
    if (!savedTaxCodes.some(e => e.taxCode === taxCode)) {
      document.getElementById("cf-result").innerHTML = `${taxCode} <button id="save-taxcode-button" class="text-button">(save)</button>`;
    } else {
      document.getElementById("cf-result").innerHTML = `${taxCode} <button id="save-taxcode-button" class="text-button" disabled>(already saved)</button>`;
    }
    
    document.getElementById("save-taxcode-button").addEventListener("click", function() {
      const entry = {
        taxCode: taxCode,
        firstName: firstName,
        lastName: lastName
      };
        
      savedTaxCodes.push(entry);
      setSavedTaxCodes(savedTaxCodes);
      document.getElementById("cf-result").innerHTML = `${taxCode} <button id="save-taxcode-button" class="text-button" disabled>(saved)</button>`;
      displaySavedTaxCodes();
    });
  }
  
  function decodeDateSexFromCode(code) {
    const year = parseInt(code.slice(0, 2), 10);
    const now = new Date();
    const pivot = now.getFullYear() % 100;
    const fullYear = year <= pivot ? 2000 + year : 1900 + year;

    const months = "ABCDEHLMPRST";
    const monthLetter = code[2];
    const month = months.indexOf(monthLetter);
    if (month === -1) {
      return null;
    }

    const daysex = parseInt(code.slice(3), 10);
    const sex = daysex > 40 ? "F" : "M";
    const day = sex === "F" ? daysex - 40 : daysex;
    
    return {
      date: new Date(fullYear, month, day),
      sex: sex
    };
  }
  
  function getSavedTaxCodesEncoded() {
    const cookieValue = document.cookie
      .split('; ')
      .find(row => row.startsWith('savedTaxCodes_v2='));
      
      if (cookieValue) {
        return cookieValue.split('=')[1];
      } else {
        return null;
      }
  }
  
  function getSavedTaxCodes() {
    const taxCodesEncoded = getSavedTaxCodesEncoded();
    
    if (!taxCodesEncoded) {
      return [];
    }
    
    const decoded = decodeURIComponent(taxCodesEncoded);
    const decoded_split = decoded.split(",");
    const entries = [];
    
    for (let i = 0; i < decoded_split.length; i += 3) {
      const chunk = decoded_split.slice(i, i + 3); 
      
      entries.push({
        taxCode: chunk[0],
        firstName: chunk[1],
        lastName: chunk[2],
      });
    }
    
    return entries;
  }
  
  function setSavedTaxCodesEncoded(taxCodesEncodedJson) {
    document.cookie = "savedTaxCodes_v2=" + taxCodesEncodedJson + "; path=/; max-age=" + (60 * 60 * 24 * 365 * 10);
  }
  
  function setSavedTaxCodes(taxCodesArray) {
    setSavedTaxCodesEncoded(encodeURIComponent(taxCodesArray.map(obj => Object.values(obj).join(","))));
  }
  
  function getExportTaxCodes() {
    const taxCodesEncodedJson = getSavedTaxCodesEncoded();
    
    if (taxCodesEncodedJson) {
      return btoa(taxCodesEncodedJson);
    } else {
      return null;
    }
  }
  
  function getExportTaxCodesUrl() {
    const exportTaxCodes = getExportTaxCodes();
    const url = new URL(window.location.href);
    
    if (exportTaxCodes) {
      url.searchParams.set("import", exportTaxCodes);
      return url.toString();
    } else {
      return null;
    }    
  }
  
  function setImportTaxCodes(importTaxCodes) {
    const taxCodesEncodedJson = atob(importTaxCodes);
    setSavedTaxCodesEncoded(taxCodesEncodedJson);
  }
  
  function deleteSavedTaxCode(taxCodeToDelete) {
    let savedTaxCodes = getSavedTaxCodes();
    savedTaxCodes = savedTaxCodes.filter(entry => entry.taxCode !== taxCodeToDelete);
    setSavedTaxCodes(savedTaxCodes);
    displaySavedTaxCodes();
    calculateTaxCode();
  }
  
  function convertSavedTaxCodeV1toV2() {
    function getSavedTaxCodesEncodedV1() {
      const cookieValue = document.cookie
        .split('; ')
        .find(row => row.startsWith('savedTaxCodes='));
      
        if (cookieValue) {
          return cookieValue.split('=')[1];
        } else {
          return null;
        }
    }
  
    function getSavedTaxCodesV1() {
      const taxCodesEncodedJson = getSavedTaxCodesEncodedV1();
  
      if (taxCodesEncodedJson) {
        try {
          return JSON.parse(decodeURIComponent(taxCodesEncodedJson));
        } catch (e) {
          return [];
        }
      } else {
        return [];
      }
    }
    
    const savedTaxCodesV1 = getSavedTaxCodesV1();
    
    if (savedTaxCodesV1.length === 0) {
      return;
    }
    
    const newEntriesV2 = [];
    
    savedTaxCodesV1.forEach(entryV1 => {
      newEntriesV2.push({
        taxCode: entryV1.taxCode,
        firstName: entryV1.firstName,
        lastName: entryV1.lastName,
      });
    });
    
    const savedTaxCodesV2 = getSavedTaxCodes();
    
    const merged = [
      ...savedTaxCodesV2,
      ...newEntriesV2.filter(e =>
        !savedTaxCodesV2.some(s => 
          s.taxCode === e.taxCode &&
          s.firstName === e.firstName &&
          s.lastName === e.lastName
        )
      )
    ];
    
    setSavedTaxCodes(merged);
    document.cookie = "savedTaxCodes=; path=/; max-age=0";
  }

  function displaySavedTaxCodes() {
    const savedTaxCodes = getSavedTaxCodes();

    if (savedTaxCodes.length === 0) {
      document.getElementById("saved-taxcodes").style.display = "none";
      return;
    }
    
    const saved_taxcodes_list_html = document.getElementById("saved-taxcodes-list");

    let html = ``;
    
    savedTaxCodes.forEach(entry => {
      const datesex = decodeDateSexFromCode(entry.taxCode.substring(6, 11));
      const formatted_date = new Intl.DateTimeFormat("it-IT", {
        day: "2-digit",
        month: "2-digit",
        year: "numeric"
      }).format(datesex.date);
    
      html += `
        <li>
          <strong>Tax code:</strong> ${entry.taxCode} <br>
          <strong>First name:</strong> ${entry.firstName} <br>
          <strong>Last name:</strong> ${entry.lastName} <br>
          <strong>Birth date:</strong> ${formatted_date} <br>
          <button data-delete-taxcode-button data-taxcode="${entry.taxCode}" id="delete-taxcode-button" class="text-button">Delete</button>
        </li>
      `;
    });
    html += "</ul>";

    saved_taxcodes_list_html.innerHTML = html;
    document.getElementById("saved-taxcodes").style.display = "block";

    document.querySelectorAll('[data-delete-taxcode-button]').forEach(button => {
      button.addEventListener("click", function() {
        const taxCodeToDelete = this.getAttribute("data-taxcode");
        deleteSavedTaxCode(taxCodeToDelete);
      });
    });
  }
  
  const inputs = document.querySelectorAll("#form-cf input");
  inputs.forEach(input => {
    input.addEventListener("input", function(event) {
      event.preventDefault();
      calculateTaxCode();
    })
  });
  
  document.getElementById("municipalityCode").addEventListener("input", (event) => {
    selectedMunicipalityCode = "";
    const inputValue = event.target.value;
    const matches = findMunicipalityCode(inputValue);
    updateSuggestions(matches);
  });

  document.addEventListener("click", (event) => {
    if (!event.target.closest("#municipalityCode")) {
      document.getElementById("municipalitySuggestions").style.display = "none";
    }
  });
  
  document.addEventListener("DOMContentLoaded", function() {
    const municipalityCode = document.getElementById("municipalityCode");
  
    loadCodiciCatastali().then(() => {
      calculateTaxCode();
    });
    
    function clearOnFirstInput(event) {
      if (municipalityCode.dataset.cleared !== "true") {
        const firstChar = event.data || municipalityCode.value.slice(-1);
        municipalityCode.value = firstChar;
        municipalityCode.dataset.cleared = "true";
        calculateTaxCode();
      }
    }

    function resetListener() {
      municipalityCode.dataset.cleared = "false";
    }
    
    municipalityCode.addEventListener("input", clearOnFirstInput);
    municipalityCode.addEventListener("blur", resetListener);
    
    if (urlParams.has("import")) {
      if (typeof import_taxcode_dialog.showModal === 'function') {
        import_taxcode_dialog.showModal();
      } else {
        import_taxcode_dialog.setAttribute('open', 'open');
      }
    }
    
    convertSavedTaxCodeV1toV2();
    displaySavedTaxCodes();
  });
  
  import_taxcode_no.addEventListener('click', () => {
    import_taxcode_dialog.close();
  });
  
  import_taxcode_yes.addEventListener('click', () => {
    const importTaxCodes = urlParams.get("import");
    setImportTaxCodes(importTaxCodes);
    displaySavedTaxCodes();
    import_taxcode_dialog.close();
  });
  
  export_taxcode_button.addEventListener('click', () => {
    const link = getExportTaxCodesUrl();
    
    if (!link) {
      return;
    }
    
    export_taxcode_link.textContent = link + ' (copy)'
    
    export_taxcode_qr.innerHTML = '';
    new QRCode(export_taxcode_qr, {
      text: link,
      width: 200,
      height: 200,
      correctLevel: QRCode.CorrectLevel.M
    });
  
    if (typeof export_taxcode_dialog.showModal === 'function') {
        export_taxcode_dialog.showModal();
      } else {
        export_taxcode_dialog.setAttribute('open', 'open');
      }
  });
  
  export_taxcode_link.addEventListener('click', async () => {
    const link = getExportTaxCodesUrl();
    
    if (!link) {
      return;
    }
    
    try {
      if (navigator.clipboard && window.isSecureContext) {
        await navigator.clipboard.writeText(link);
      } else {
        const ta = document.createElement('textarea');
        ta.value = link;
        document.body.appendChild(ta);
        ta.select();
        document.execCommand('copy');
        ta.remove();
      }
      
      export_taxcode_link.textContent = link + ' (copied)'
      setTimeout(() => (export_taxcode_link.textContent = link + ' (copy)'), 1500);
    } catch (e) {
      console.log(e);
    }
  });
  
  export_taxcode_close.addEventListener('click', () => {
    export_taxcode_dialog.close();
  });
  
  export_taxcode_dialog.addEventListener('click', (e) => {
    const rect = export_taxcode_dialog.getBoundingClientRect();
    const inDialog = (
      e.clientX >= rect.left && e.clientX <= rect.right &&
      e.clientY >= rect.top  && e.clientY <= rect.bottom
    );
    if (!inDialog) export_taxcode_dialog.close();
  });
</script>
