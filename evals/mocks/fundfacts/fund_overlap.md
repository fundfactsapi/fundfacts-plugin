---
expect:
  isins: array
---
{"funds":[{"isin":"IE00B4L5Y983","name":"iShares Core MSCI World UCITS ETF","holdings":10,"holdingsBasis":"portfolio","weighted":true},{"isin":"IE000EXMPL24","name":"Example FTSE All-World UCITS ETF (Dist)","holdings":10,"holdingsBasis":"portfolio","weighted":true}],"pairs":[{"a":"IE00B4L5Y983","b":"IE000EXMPL24","overlap":21.6,"sharedCount":9,"shared":[{"label":"NVIDIA","a":5.48,"b":4.61},{"label":"APPLE","a":5.24,"b":4.42},{"label":"MICROSOFT","a":3.88,"b":3.27}],"sharedNames":["NVIDIA","APPLE","MICROSOFT"],"disclosed":{"a":26.7,"b":22.9},"comparable":true}],"excluded":[],"namesOnly":[],"note":"Overlap is computed over the holdings each fund discloses and matched by issuer name; funds that publish only their top ten will show a lower bound.","rules":"fundfacts-portfolio/3","pending":[],"notFound":[]}
