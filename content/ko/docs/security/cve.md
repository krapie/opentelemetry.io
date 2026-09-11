---
title: 일반 취약점 및 노출
weight: 100
default_lang_commit: 58e684763e8dd50a07ef5fbf428303973025428a
---

이 페이지는
[깃허브(GitHub)의 오픈텔레메트리(OpenTelemetry) 조직](https://github.com/open-telemetry/)
내 모든 저장소에서 보고된 일반 취약점 및 노출(Common Vulnerabilities and
Exposures, CVE) 목록이다. 원본 데이터는
[sig-security](https://github.com/open-telemetry/sig-security) 저장소에 저장되어
있으며, 매일 갱신된다.

<table id="cve-table">
  <thead>
    <tr>
      <th>CVE ID</th>
      <th>Issue Summary</th>
      <th>Severity</th>
      <th>Repository</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>

<!-- markdownlint-disable no-shortcut-ref-link -->

<script id="main-script">
  'use strict';
  (function() {
    function fetchAndRender() {
      fetchData()
        .then(renderTable);
    }

    function fetchData() {
      var url = 'https://raw.githubusercontent.com/open-telemetry/sig-security/data-source/published_output.json';
      return fetch(url)
        .then(function(response) {
          return response.json();
        });
    }

    function renderTable(data) {
      var table = document.getElementById('cve-table').querySelector('tbody');

      data.sort((a, b) => b.cve_id.localeCompare(a.cve_id));

      data.forEach(item => {
        var row = table.insertRow();

        const cell1 = row.insertCell(0);
        const link = document.createElement('a');
        link.href = item['html_url'];
        link.target = '_blank';
        link.textContent = item['cve_id'];
        cell1.appendChild(link);

        const cell2 = row.insertCell(1);
        cell2.textContent = item['summary'];
        const cell3 = row.insertCell(2);
        cell3.textContent = item['severity'];

        const cell4 = row.insertCell(3);
        // cell4.textContent = item['repo'];
        const link2 = document.createElement('a');
        link2.href = 'https://www.github.com/open-telemetry/' + item['repo'] + '/security/advisories';
        link2.target = '_blank';
        link2.textContent = item['repo'];
        cell4.appendChild(link2);
      });
    }

    fetchAndRender();
  })();
</script>
