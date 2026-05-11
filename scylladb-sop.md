<img width="124" height="150" alt="scylladb_cluster_architecture" src="https://github.com/user-attachments/assets/f6482502-798f-4aac-bb97-7ec2d3d0c692" />
# ScyllaDB 3-Node Cluster — SOP

**OS:** Rocky Linux | **Version:** ScyllaDB 6.2.3
**Nodes:** `.34` · `.35` · `.237`

---<svg width="100%" viewBox="0 0 680 820" role="img" style="" xmlns="http://www.w3.org/2000/svg">
  <title style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">ScyllaDB 3-node cluster architecture</title>
  <desc style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">Shows the consistent hash ring, token ranges, partition key flow, and replication across all 3 nodes</desc>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  <mask id="imagine-text-gaps-xt5au3" maskUnits="userSpaceOnUse"><rect x="0" y="0" width="680" height="820" fill="white"/><rect x="211.04971313476562" y="16.363636016845703" width="258.17430114746094" height="21.27272605895996" fill="black" rx="2"/><rect x="313.7485656738281" y="57.3636360168457" width="52.502838134765625" height="21.27272605895996" fill="black" rx="2"/><rect x="297.4630432128906" y="76.7272720336914" width="85.07386016845703" height="18.545454025268555" fill="black" rx="2"/><rect x="494.3352355957031" y="283.3636169433594" width="55.32954406738281" height="21.27272605895996" fill="black" rx="2"/><rect x="479.7613525390625" y="302.7272644042969" width="84.4772720336914" height="18.545454025268555" fill="black" rx="2"/><rect x="130.49147033691406" y="283.3636169433594" width="55.280426025390625" height="21.27272605895996" fill="black" rx="2"/><rect x="112.57955932617188" y="302.7272644042969" width="90.84090423583984" height="18.545454025268555" fill="black" rx="2"/><rect x="391.9516906738281" y="136.1818084716797" width="76.20999908447266" height="18.545454025268555" fill="black" rx="2"/><rect x="330.3252868652344" y="316.1817932128906" width="79.34942626953125" height="18.545454025268555" fill="black" rx="2"/><rect x="173.03834533691406" y="136.1818084716797" width="73.92329406738281" height="18.545454025268555" fill="black" rx="2"/><rect x="303.75567626953125" y="181.1818084716797" width="72.68794250488281" height="18.545454025268555" fill="black" rx="2"/><rect x="310.86505126953125" y="199.1818084716797" width="58.587379455566406" height="19.45454502105713" fill="black" rx="2"/><rect x="271.234375" y="217.1818084716797" width="137.53125" height="19.45454502105713" fill="black" rx="2"/><rect x="265.75140380859375" y="374.3636169433594" width="148.4971466064453" height="21.27272605895996" fill="black" rx="2"/><rect x="78.21590423583984" y="422.3636474609375" width="63.61143112182617" height="21.27272605895996" fill="black" rx="2"/><rect x="61.33380126953125" y="441.7272644042969" width="97.33238220214844" height="19.45454502105713" fill="black" rx="2"/><rect x="250.1832275390625" y="422.3636474609375" width="69.90831756591797" height="21.27272605895996" fill="black" rx="2"/><rect x="242.18606567382812" y="441.7272644042969" width="85.62783813476562" height="18.545454025268555" fill="black" rx="2"/><rect x="426.49005126953125" y="422.3636474609375" width="47.0198860168457" height="21.27272605895996" fill="black" rx="2"/><rect x="416.02838134765625" y="441.7272644042969" width="67.94318008422852" height="18.545454025268555" fill="black" rx="2"/><rect x="577.9104614257812" y="422.3636474609375" width="44.17897415161133" height="21.27272605895996" fill="black" rx="2"/><rect x="563.8338012695312" y="441.7272644042969" width="72.66130828857422" height="18.545454025268555" fill="black" rx="2"/><rect x="207.18606567382812" y="486.3636169433594" width="265.6278381347656" height="21.27272605895996" fill="black" rx="2"/><rect x="237.05113220214844" y="506.1817932128906" width="206.10104370117188" height="19.45454502105713" fill="black" rx="2"/><rect x="62.78266906738281" y="549.3635864257812" width="124.47425842285156" height="21.27272605895996" fill="black" rx="2"/><rect x="71.58948516845703" y="568.7272338867188" width="106.8210220336914" height="18.545454025268555" fill="black" rx="2"/><rect x="86.5397720336914" y="586.727294921875" width="76.92044830322266" height="19.45454502105713" fill="black" rx="2"/><rect x="279.6647644042969" y="549.3635864257812" width="121.09536743164062" height="21.27272605895996" fill="black" rx="2"/><rect x="286.1846618652344" y="568.7272338867188" width="108.05146026611328" height="18.545454025268555" fill="black" rx="2"/><rect x="300.28265380859375" y="586.727294921875" width="79.43465423583984" height="19.45454502105713" fill="black" rx="2"/><rect x="494.8210144042969" y="549.3635864257812" width="120.78053283691406" height="21.27272605895996" fill="black" rx="2"/><rect x="498.0099182128906" y="568.7272338867188" width="114.4111328125" height="18.545454025268555" fill="black" rx="2"/><rect x="515.4246826171875" y="586.727294921875" width="79.15056610107422" height="19.45454502105713" fill="black" rx="2"/><rect x="224.92755126953125" y="632.3635864257812" width="230.2061309814453" height="21.27272605895996" fill="black" rx="2"/><rect x="175.39630126953125" y="652.1818237304688" width="329.2073669433594" height="19.45454502105713" fill="black" rx="2"/><rect x="75.73011016845703" y="692.3635864257812" width="58.539772033691406" height="21.27272605895996" fill="black" rx="2"/><rect x="69.57244110107422" y="709.727294921875" width="70.88164901733398" height="18.545454025268555" fill="black" rx="2"/><rect x="222.2783966064453" y="692.3635864257812" width="55.443180084228516" height="21.27272605895996" fill="black" rx="2"/><rect x="215.97158813476562" y="709.727294921875" width="68.05681610107422" height="18.545454025268555" fill="black" rx="2"/><rect x="365.86505126953125" y="692.3635864257812" width="58.2698860168457" height="21.27272605895996" fill="black" rx="2"/><rect x="359.7144775390625" y="709.727294921875" width="70.5710220336914" height="18.545454025268555" fill="black" rx="2"/><rect x="511.02838134765625" y="692.3635864257812" width="58.2186164855957" height="21.27272605895996" fill="black" rx="2"/><rect x="504.85650634765625" y="709.727294921875" width="70.28693008422852" height="18.545454025268555" fill="black" rx="2"/><rect x="51.09232711791992" y="729.1817626953125" width="108.24711608886719" height="18.545454025268555" fill="black" rx="2"/><rect x="196.09231567382812" y="729.1817626953125" width="108.24713134765625" height="18.545454025268555" fill="black" rx="2"/><rect x="341.0923156738281" y="729.1817626953125" width="108.24713134765625" height="18.545454025268555" fill="black" rx="2"/><rect x="486.0923156738281" y="729.1817626953125" width="108.24713134765625" height="18.545454025268555" fill="black" rx="2"/><rect x="54" y="763.1818237304688" width="80.52002716064453" height="18.545454025268555" fill="black" rx="2"/><rect x="174" y="763.1818237304688" width="83.03160858154297" height="18.545454025268555" fill="black" rx="2"/><rect x="294" y="763.1818237304688" width="82.743896484375" height="18.545454025268555" fill="black" rx="2"/><rect x="416" y="763.1818237304688" width="157.70191955566406" height="19.45454502105713" fill="black" rx="2"/></mask></defs>

  <!-- ── TITLE ── -->
  <text x="340" y="32" text-anchor="middle" style="fill:rgb(250, 249, 245);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:auto">Consistent hash ring — 3 nodes, RF=3</text>

  <!-- ── RING ── -->
  <circle cx="340" cy="210" r="130" fill="none" stroke="var(--b)" stroke-width="1" stroke-dasharray="4 3" opacity="0.4" style="fill:none;stroke:rgba(222, 220, 209, 0.3);color:rgb(255, 255, 255);stroke-width:1px;stroke-dasharray:4px, 3px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.4;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- Token arc segments (3 x 120°) -->
  <!-- Node 1 arc: top → right (0°→120°) -->
  <path d="M340 80 A130 130 0 0 1 452.6 275" fill="none" stroke="#534AB7" stroke-width="5" stroke-linecap="round" opacity="0.85" mask="url(#imagine-text-gaps-xt5au3)" style="fill:none;stroke:rgb(83, 74, 183);color:rgb(255, 255, 255);stroke-width:5px;stroke-linecap:round;stroke-linejoin:miter;opacity:0.85;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <!-- Node 2 arc: right → bottom-left (120°→240°) -->
  <path d="M452.6 275 A130 130 0 0 1 227.4 275" fill="none" stroke="#0F6E56" stroke-width="5" stroke-linecap="round" opacity="0.85" mask="url(#imagine-text-gaps-xt5au3)" style="fill:none;stroke:rgb(15, 110, 86);color:rgb(255, 255, 255);stroke-width:5px;stroke-linecap:round;stroke-linejoin:miter;opacity:0.85;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <!-- Node 3 arc: bottom-left → top (240°→360°) -->
  <path d="M227.4 275 A130 130 0 0 1 340 80" fill="none" stroke="#BA7517" stroke-width="5" stroke-linecap="round" opacity="0.85" mask="url(#imagine-text-gaps-xt5au3)" style="fill:none;stroke:rgb(186, 117, 23);color:rgb(255, 255, 255);stroke-width:5px;stroke-linecap:round;stroke-linejoin:miter;opacity:0.85;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- Node 1 — top (0°) -->
  <g onclick="sendPrompt('What token range does Node 1 own in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="270" y="48" width="140" height="52" rx="8" stroke-width="0.5" style="fill:rgb(60, 52, 137);stroke:rgb(175, 169, 236);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="340" y="68" text-anchor="middle" dominant-baseline="central" style="fill:rgb(206, 203, 246);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 1</text>
    <text x="340" y="86" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">192.168.122.34</text>
  </g>

  <!-- Node 2 — bottom right (120°) -->
  <g onclick="sendPrompt('What token range does Node 2 own in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="452" y="274" width="140" height="52" rx="8" stroke-width="0.5" style="fill:rgb(8, 80, 65);stroke:rgb(93, 202, 165);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="522" y="294" text-anchor="middle" dominant-baseline="central" style="fill:rgb(159, 225, 203);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 2</text>
    <text x="522" y="312" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">192.168.122.35</text>
  </g>

  <!-- Node 3 — bottom left (240°) -->
  <g onclick="sendPrompt('What token range does Node 3 own in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="88" y="274" width="140" height="52" rx="8" stroke-width="0.5" style="fill:rgb(99, 56, 6);stroke:rgb(239, 159, 39);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="158" y="294" text-anchor="middle" dominant-baseline="central" style="fill:rgb(250, 199, 117);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 3</text>
    <text x="158" y="312" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">192.168.122.237</text>
  </g>

  <!-- Token range labels on arcs -->
  <text x="430" y="150" text-anchor="middle" opacity="0.7" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.7;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">tokens 0→⅓</text>
  <text x="370" y="330" text-anchor="middle" opacity="0.7" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.7;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">tokens ⅓→⅔</text>
  <text x="210" y="150" text-anchor="middle" opacity="0.7" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.7;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">tokens ⅔→1</text>

  <!-- Shard count indicator inside ring -->
  <text x="340" y="195" text-anchor="middle" opacity="0.55" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.55;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">256 vnodes</text>
  <text x="340" y="213" text-anchor="middle" opacity="0.55" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.55;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">per node</text>
  <text x="340" y="231" text-anchor="middle" opacity="0.35" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.35;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">(random token spread)</text>

  <!-- ══════════════════════════════════════════
       PARTITION KEY FLOW
  ══════════════════════════════════════════ -->
  <text x="340" y="390" text-anchor="middle" style="fill:rgb(250, 249, 245);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:auto">How a write is routed</text>

  <!-- Step 1: Row key -->
  <g onclick="sendPrompt('What is a partition key in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="40" y="415" width="140" height="48" rx="8" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="110" y="433" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Row key</text>
    <text x="110" y="451" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">e.g. employee id</text>
  </g>

  <!-- arrow -->
  <line x1="182" y1="439" x2="218" y2="439" marker-end="url(#arrow)" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- Step 2: Murmur3 hash -->
  <g onclick="sendPrompt('How does Murmur3 hashing work in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="220" y="415" width="130" height="48" rx="8" stroke-width="0.5" style="fill:rgb(60, 52, 137);stroke:rgb(175, 169, 236);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="285" y="433" text-anchor="middle" dominant-baseline="central" style="fill:rgb(206, 203, 246);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Murmur3</text>
    <text x="285" y="451" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">hash function</text>
  </g>

  <!-- arrow -->
  <line x1="352" y1="439" x2="388" y2="439" marker-end="url(#arrow)" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- Step 3: Token -->
  <g onclick="sendPrompt('What is a token in ScyllaDB consistent hashing?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="390" y="415" width="120" height="48" rx="8" stroke-width="0.5" style="fill:rgb(99, 56, 6);stroke:rgb(239, 159, 39);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="450" y="433" text-anchor="middle" dominant-baseline="central" style="fill:rgb(250, 199, 117);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Token</text>
    <text x="450" y="451" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">-2⁶³ to +2⁶³</text>
  </g>

  <!-- arrow -->
  <line x1="512" y1="439" x2="548" y2="439" marker-end="url(#arrow)" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- Step 4: Coordinator -->
  <g onclick="sendPrompt('What does the coordinator node do in ScyllaDB?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="550" y="415" width="100" height="48" rx="8" stroke-width="0.5" style="fill:rgb(8, 80, 65);stroke:rgb(93, 202, 165);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="600" y="433" text-anchor="middle" dominant-baseline="central" style="fill:rgb(159, 225, 203);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node</text>
    <text x="600" y="451" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">owns range</text>
  </g>

  <!-- ══════════════════════════════════════════
       REPLICATION — RF=3
  ══════════════════════════════════════════ -->
  <text x="340" y="502" text-anchor="middle" style="fill:rgb(250, 249, 245);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:auto">Replication factor = 3 (SimpleStrategy)</text>
  <text x="340" y="520" text-anchor="middle" opacity="0.6" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.6;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">Every row is written to ALL 3 nodes</text>

  <!-- 3 replica boxes side by side -->
  <!-- Node 1 replica -->
  <g onclick="sendPrompt('What does replication factor 3 mean for data safety?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="40" y="540" width="170" height="76" rx="8" stroke-width="0.5" style="fill:rgb(60, 52, 137);stroke:rgb(175, 169, 236);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="125" y="560" text-anchor="middle" dominant-baseline="central" style="fill:rgb(206, 203, 246);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 1 — primary</text>
    <text x="125" y="578" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">.34 — owns token</text>
    <text x="125" y="596" text-anchor="middle" dominant-baseline="central" style="fill:rgb(175, 169, 236);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">replica 1 of 3</text>
  </g>

  <!-- Node 2 replica -->
  <g onclick="sendPrompt('How does ScyllaDB decide which node gets replica 2?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="255" y="540" width="170" height="76" rx="8" stroke-width="0.5" style="fill:rgb(8, 80, 65);stroke:rgb(93, 202, 165);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="340" y="560" text-anchor="middle" dominant-baseline="central" style="fill:rgb(159, 225, 203);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 2 — replica</text>
    <text x="340" y="578" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">.35 — next on ring</text>
    <text x="340" y="596" text-anchor="middle" dominant-baseline="central" style="fill:rgb(93, 202, 165);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">replica 2 of 3</text>
  </g>

  <!-- Node 3 replica -->
  <g onclick="sendPrompt('What happens if one node goes down with RF=3?')" style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="470" y="540" width="170" height="76" rx="8" stroke-width="0.5" style="fill:rgb(99, 56, 6);stroke:rgb(239, 159, 39);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="555" y="560" text-anchor="middle" dominant-baseline="central" style="fill:rgb(250, 199, 117);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Node 3 — replica</text>
    <text x="555" y="578" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">.237 — next on ring</text>
    <text x="555" y="596" text-anchor="middle" dominant-baseline="central" style="fill:rgb(239, 159, 39);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">replica 3 of 3</text>
  </g>

  <!-- Write arrows from coordinator down to replicas -->
  <line x1="125" y1="463" x2="125" y2="538" marker-end="url(#arrow)" stroke="#534AB7" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <line x1="600" y1="463" x2="340" y2="538" marker-end="url(#arrow)" stroke="#0F6E56" mask="url(#imagine-text-gaps-xt5au3)" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <line x1="600" y1="463" x2="555" y2="538" marker-end="url(#arrow)" stroke="#BA7517" style="fill:none;stroke:rgb(156, 154, 146);color:rgb(255, 255, 255);stroke-width:1.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>

  <!-- ══════════════════════════════════════════
       SHARDING — per node internals
  ══════════════════════════════════════════ -->
  <text x="340" y="648" text-anchor="middle" style="fill:rgb(250, 249, 245);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:auto">Inside each node — CPU sharding</text>
  <text x="340" y="666" text-anchor="middle" opacity="0.6" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.6;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">ScyllaDB assigns one shard per CPU core (share-nothing)</text>

  <!-- Shard boxes — 4 shards representing cores -->
  <g style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="40" y="685" width="130" height="44" rx="6" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="105" y="703" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Shard 0</text>
    <text x="105" y="719" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">CPU core 0</text>
  </g>
  <g style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="185" y="685" width="130" height="44" rx="6" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="250" y="703" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Shard 1</text>
    <text x="250" y="719" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">CPU core 1</text>
  </g>
  <g style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="330" y="685" width="130" height="44" rx="6" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="395" y="703" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Shard 2</text>
    <text x="395" y="719" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">CPU core 2</text>
  </g>
  <g style="fill:rgb(0, 0, 0);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto">
    <rect x="475" y="685" width="130" height="44" rx="6" stroke-width="0.5" style="fill:rgb(68, 68, 65);stroke:rgb(180, 178, 169);color:rgb(255, 255, 255);stroke-width:0.5px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
    <text x="540" y="703" text-anchor="middle" dominant-baseline="central" style="fill:rgb(211, 209, 199);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:14px;font-weight:500;text-anchor:middle;dominant-baseline:central">Shard 3</text>
    <text x="540" y="719" text-anchor="middle" dominant-baseline="central" style="fill:rgb(180, 178, 169);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:central">CPU core 3</text>
  </g>

  <!-- Shard sub-labels -->
  <text x="105" y="743" text-anchor="middle" opacity="0.5" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.5;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">own token subset</text>
  <text x="250" y="743" text-anchor="middle" opacity="0.5" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.5;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">own token subset</text>
  <text x="395" y="743" text-anchor="middle" opacity="0.5" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.5;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">own token subset</text>
  <text x="540" y="743" text-anchor="middle" opacity="0.5" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.5;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:middle;dominant-baseline:auto">own token subset</text>

  <!-- Legend -->
  <rect x="40" y="766" width="12" height="12" rx="2" fill="#534AB7" opacity="0.8" style="fill:rgb(83, 74, 183);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.8;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="58" y="777" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:start;dominant-baseline:auto">Node 1 range</text>
  <rect x="160" y="766" width="12" height="12" rx="2" fill="#0F6E56" opacity="0.8" style="fill:rgb(15, 110, 86);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.8;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="178" y="777" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:start;dominant-baseline:auto">Node 2 range</text>
  <rect x="280" y="766" width="12" height="12" rx="2" fill="#BA7517" opacity="0.8" style="fill:rgb(186, 117, 23);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.8;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:16px;font-weight:400;text-anchor:start;dominant-baseline:auto"/>
  <text x="298" y="777" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:1;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:start;dominant-baseline:auto">Node 3 range</text>
  <text x="420" y="777" opacity="0.55" style="fill:rgb(194, 192, 182);stroke:none;color:rgb(255, 255, 255);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;opacity:0.55;font-family:&quot;Anthropic Sans&quot;, -apple-system, BlinkMacSystemFont, &quot;Segoe UI&quot;, sans-serif;font-size:12px;font-weight:400;text-anchor:start;dominant-baseline:auto">click any box to learn more</text>
</svg>

## 1. Pre-installation

- Verify CPU supports SSE4.2 + PCLMUL on all nodes:
  ```bash
  lscpu | grep -E 'sse4_2|pclmul'
  ```
- Update system:
  ```bash
  sudo dnf update -y
  sudo dnf install -y curl gnupg2 net-tools
  ```

---

## 2. Install ScyllaDB

Run on **all 3 nodes**:

```bash
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-version 6.2.3
```

---

## 3. Configure `/etc/scylla/scylla.yaml`

Edit on **each node** — change `listen_address` and `rpc_address` per node:

```yaml
cluster_name: 'Scylla-Cluster'

data_file_directories:
    - /var/lib/scylla/data
commitlog_directory: /var/lib/scylla/commitlog

seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "192.168.122.34,192.168.122.35,192.168.122.237"

listen_address: 192.168.122.XX      # this node's IP
rpc_address: 192.168.122.XX         # this node's IP

endpoint_snitch: GossipingPropertyFileSnitch
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

| Node   | `listen_address` / `rpc_address` |
|--------|----------------------------------|
| Node 1 | `192.168.122.34`                 |
| Node 2 | `192.168.122.35`                 |
| Node 3 | `192.168.122.237`                |

---

## 4. Configure `/etc/scylla/cassandra-rackdc.properties`

Same on all 3 nodes:

```properties
dc=scylla_data_center
rack=scylla_rack
```

---

## 5. Open firewall ports (Rocky Linux — all nodes)

```bash
sudo firewall-cmd --permanent --add-port=7000/tcp   # inter-node
sudo firewall-cmd --permanent --add-port=9042/tcp   # CQL clients
sudo firewall-cmd --permanent --add-port=10000/tcp  # REST API
sudo firewall-cmd --reload
```

---

## 6. SELinux (Rocky Linux)

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

---

## 7. Run system optimization

```bash
sudo scylla_setup
```

Key responses:

| Prompt                     | Answer |
|----------------------------|--------|
| Check kernel version?      | yes    |
| Auto-start on boot?        | yes    |
| Setup NTP?                 | yes    |
| Setup RAID and XFS?        | no     |
| Run iotune?                | yes    |
| CPU scaling governor?      | yes    |
| Enable fstrim service?     | yes    |
| Scylla only service?       | yes    |
| Tune LimitNOFILES?         | yes    |

---

## 8. Start service (all nodes)

```bash
sudo systemctl start scylla-server
sudo systemctl enable scylla-server
sudo systemctl status scylla-server
```

Wait ~2 min, then verify:

```bash
nodetool status
```

Expected — all 3 rows show `UN`:

```
Datacenter: scylla_data_center
================================
-- Address           Tokens  Status
UN 192.168.122.34    256     Up/Normal
UN 192.168.122.35    256     Up/Normal
UN 192.168.122.237   256     Up/Normal
```

---

## 9. Security setup

Connect:

```bash
cqlsh -u cassandra -p cassandra 192.168.122.34 9042
```

```sql
-- Change default password immediately
ALTER ROLE cassandra WITH PASSWORD = 'your-secure-password';

-- Create admin user
CREATE ROLE dbadmin WITH SUPERUSER = true AND LOGIN = true AND PASSWORD = 'admin-password';

-- Verify
LIST ROLES;
```

---

## 10. Create keyspace & table

```sql
CREATE KEYSPACE test_keyspace
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
} AND durable_writes = true;

USE test_keyspace;

CREATE TABLE employees (
    id UUID PRIMARY KEY,
    name TEXT,
    department TEXT,
    salary INT,
    joined_date DATE
);
```

---

## 11. How data flows

### INSERT

```
Client
  |
  | INSERT INTO employees VALUES (uuid(), 'Alice', ...)
  v
Murmur3 hash(uuid)  -->  token = -6,234,891,203,...
  |
  v
Token ring lookup
  [Node1: 0→1/3] [Node2: 1/3→2/3 <-- token lands here] [Node3: 2/3→1]
  |
  v
Node2 = coordinator (owns the token)
  |
  |-- write --> Node1 (.34)  [replica]
  |-- write --> Node2 (.35)  [primary]
  |-- write --> Node3 (.237) [replica]
  |
  v
All 3 write to commitlog + memtable --> ACK to client
```

### SELECT

```
Client
  |
  | SELECT * FROM employees WHERE id = <uuid>
  v
Murmur3 hash(same uuid)  -->  same token --> Node2 = coordinator
  |
  |-- read --> Node1 (.34)   responds with row + timestamp
  |-- read --> Node2 (.35)   responds with row + timestamp
  |            Node3 (.237)  NOT contacted (QUORUM = 2 of 3)
  v
Coordinator picks newest timestamp --> returns row to client
```

> If a node is down, coordinator reads from the remaining 2. Data is safe.

### vnode explained

```
Token space:  |----0-----------------------------------------2^63----|

Node1 owns 256 small scattered segments across the ring:
  |--N1--|--N2--|--N3--|--N1--|--N3--|--N2--|--N1--|--N2--|--N3--|

Not one big chunk — 256 small ones = balanced load
```

---

## 12. Verify replication

After inserting data on `.34`, query from `.35` and `.237`:

```bash
cqlsh -u cassandra -p cassandra 192.168.122.35 9042 \
  -e "SELECT * FROM test_keyspace.employees;"

cqlsh -u cassandra -p cassandra 192.168.122.237 9042 \
  -e "SELECT * FROM test_keyspace.employees;"
```

All 3 must return identical rows.

---

## 13. Common maintenance commands

```bash
nodetool status          # cluster health
nodetool info            # node details
nodetool repair          # fix data consistency
nodetool cleanup         # after adding/removing nodes
nodetool flush           # flush memtables to disk
nodetool snapshot <ks>   # backup keyspace
nodetool listsnapshots   # list backups
```

---

## 14. Troubleshooting

| Symptom | Check |
|---------|-------|
| Node missing from `nodetool status` | `journalctl -u scylla-server -n 100` |
| Port 9042 refused | `firewall-cmd --list-ports` |
| Node stuck joining | wipe `/var/lib/scylla/data/*` and restart |
| Auth error on ALTER ROLE | verify `authenticator` + `authorizer` in yaml, restart all nodes |
| SELinux blocking | `sudo setenforce 0` then restart |

### Node not joining — clean bootstrap

```bash
sudo systemctl stop scylla-server
sudo rm -rf /var/lib/scylla/data/*
sudo rm -rf /var/lib/scylla/commitlog/*
sudo rm -rf /var/lib/scylla/hints/*
sudo rm -rf /var/lib/scylla/view_hints/*
sudo systemctl start scylla-server
sudo journalctl -u scylla-server -f   # watch for "entering NORMAL mode"
```
