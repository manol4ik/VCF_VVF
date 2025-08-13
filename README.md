# VCF_VVF
pequeña calculadora
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>VCF/VVF Auto-Sizing — cores físicos vs. suscritos</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- React 18 UMD -->
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <!-- Babel para permitir JSX en una sola página (más simple para GitHub Pages) -->
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-white text-gray-900">
  <div class="max-w-6xl mx-auto p-6" id="root"></div>

  <script type="text/babel">
    const { useMemo, useState } = React;

    function App() {
      // ---- Inputs (estado) ----
      const [minCoresPerCpu, setMinCoresPerCpu] = useState(16);
      const [vvfGiBPerCore, setVvfGiBPerCore] = useState(250); // VVF incluye 250 GiB/core
      const [vcfTiBPerCore, setVcfTiBPerCore] = useState(1);   // VCF incluye 1 TiB/core
      const [vsanRequiredTiB, setVsanRequiredTiB] = useState(200);

      // Add-ons
      const [firewallCoresOverride, setFirewallCoresOverride] = useState(0); // si 0 -> usa cores suscritos
      const [liveRecoveryVMs, setLiveRecoveryVMs] = useState(0);
      const [liveCyberRecoveryVMs, setLiveCyberRecoveryVMs] = useState(0);
      const [liveCyberRecoveryTiB, setLiveCyberRecoveryTiB] = useState(0);

      // Inventario (precargado con tu caso)
      const [rows, setRows] = useState([
        { id: 1, name: "Grupo A", servers: 16, cpusPerServer: 2, coresPerCpu: 20 },
        { id: 2, name: "Grupo B", servers: 8,  cpusPerServer: 1, coresPerCpu: 8 },
      ]);

      const addRow = () => {
        const nextId = (rows.at(-1)?.id ?? 0) + 1;
        setRows([...rows, { id: nextId, name: `Grupo ${String.fromCharCode(64 + ((nextId-1)%26)+1)}` , servers: 0, cpusPerServer: 0, coresPerCpu: 0 }]);
      };

      const resetExample = () => {
        setMinCoresPerCpu(16);
        setVvfGiBPerCore(250);
        setVcfTiBPerCore(1);
        setVsanRequiredTiB(200);
        setFirewallCoresOverride(0);
        setLiveRecoveryVMs(0);
        setLiveCyberRecoveryVMs(0);
        setLiveCyberRecoveryTiB(0);
        setRows([
          { id: 1, name: "Grupo A", servers: 16, cpusPerServer: 2, coresPerCpu: 20 },
          { id: 2, name: "Grupo B", servers: 8,  cpusPerServer: 1, coresPerCpu: 8 },
        ]);
      };

      // ---- Cálculos ----
      const calc = useMemo(() => {
        const details = rows.map((r) => {
          const servers = Number(r.servers) || 0;
          const cpusPerServer = Number(r.cpusPerServer) || 0;
          const coresPerCpu = Number(r.coresPerCpu) || 0;
          const totalCpus = servers * cpusPerServer;
          const physicalCores = totalCpus * coresPerCpu;
          const subscribedPerCpu = Math.max(coresPerCpu, Number(minCoresPerCpu) || 0);
          const subscribedCores = totalCpus * subscribedPerCpu;
          const delta = subscribedCores - physicalCores;
          return {
            ...r,
            totalCpus,
            physicalCores,
            subscribedPerCpu,
            subscribedCores,
            delta,
          };
        });

        const totalCpus = details.reduce((s, d) => s + d.totalCpus, 0);
        const totalPhysical = details.reduce((s, d) => s + d.physicalCores, 0);
        const totalSubscribed = details.reduce((s, d) => s + d.subscribedCores, 0);
        const totalDelta = totalSubscribed - totalPhysical;

        // vSAN incluida
        const includedVvfTiB = (Number(vvfGiBPerCore) * totalSubscribed) / 1024; // 1 TiB = 1024 GiB
        const includedVcfTiB = Number(vcfTiBPerCore) * totalSubscribed;

        const requiredTiB = Number(vsanRequiredTiB) || 0;
        const addVvfTiB = Math.max(requiredTiB - includedVvfTiB, 0);
        const addVcfTiB = Math.max(requiredTiB - includedVcfTiB, 0);

        const firewallCores = Number(firewallCoresOverride) > 0 ? Number(firewallCoresOverride) : totalSubscribed;

        return {
          details,
          totalCpus,
          totalPhysical,
          totalSubscribed,
          totalDelta,
          includedVvfTiB,
          includedVcfTiB,
          addVvfTiB,
          addVcfTiB,
          firewallCores,
          liveRecoveryVMs: Number(liveRecoveryVMs) || 0,
          liveCyberRecoveryVMs: Number(liveCyberRecoveryVMs) || 0,
          liveCyberRecoveryTiB: Number(liveCyberRecoveryTiB) || 0,
        };
      }, [rows, minCoresPerCpu, vvfGiBPerCore, vcfTiBPerCore, vsanRequiredTiB, firewallCoresOverride, liveRecoveryVMs, liveCyberRecoveryVMs, liveCyberRecoveryTiB]);

      const exportCsv = () => {
        const lines = [];
        lines.push(["Grupo","Servidores","CPUs/Servidor","Cores/CPU (físicos)","CPUs totales","Cores físicos","Cores suscritos/CPU","Cores suscritos","Delta"].join(","));
        calc.details.forEach(d => {
          lines.push([d.name,d.servers,d.cpusPerServer,d.coresPerCpu,d.totalCpus,d.physicalCores,d.subscribedPerCpu,d.subscribedCores,d.delta].join(","));
        });
        lines.push("");
        lines.push(["CPUs totales",calc.totalCpus].join(","));
        lines.push(["Cores físicos totales",calc.totalPhysical].join(","));
        lines.push(["Cores suscritos totales",calc.totalSubscribed].join(","));
        lines.push(["Delta suscrito − físico",calc.totalDelta].join(","));
        lines.push("");
        lines.push(["vSAN requerida (TiB)", vsanRequiredTiB].join(","));
        lines.push(["Incluido VVF (TiB)", calc.includedVvfTiB.toFixed(2)].join(","));
        lines.push(["Incluido VCF (TiB)", calc.includedVcfTiB.toFixed(2)].join(","));
        lines.push(["Adicional VVF (TiB)", calc.addVvfTiB.toFixed(2)].join(","));
        lines.push(["Adicional VCF (TiB)", calc.addVcfTiB.toFixed(2)].join(","));
        lines.push("");
        lines.push(["Firewall cores", calc.firewallCores].join(","));
        lines.push(["Live Recovery VMs", calc.liveRecoveryVMs].join(","));
        lines.push(["Live Cyber Recovery VMs", calc.liveCyberRecoveryVMs].join(","));
        lines.push(["Live Cyber Recovery Capacidad (TiB)", calc.liveCyberRecoveryTiB].join(","));

        const csv = lines.join("\n");
        const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "VCF_VVF_Sizing_Export.csv";
        a.click();
        URL.revokeObjectURL(url);
      };

      return (
        <div className="space-y-6">
          <header className="space-y-1">
            <h1 className="text-2xl font-bold">VCF/VVF Auto-Sizing — cores físicos vs. suscritos</h1>
            <p className="text-sm text-gray-600">Reglas: mínimo {minCoresPerCpu} cores/CPU para suscripción; VVF incluye {vvfGiBPerCore} GiB/core; VCF incluye {vcfTiBPerCore} TiB/core. 1 TiB = 1024 GiB.</p>
          </header>

          {/* Parámetros generales */}
          <section className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <div className="flex flex-col">
              <label className="text-sm font-semibold">Mínimo cores por CPU</label>
              <input type="number" className="border rounded p-2" value={minCoresPerCpu} onChange={e=>setMinCoresPerCpu(parseInt(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">VVF (GiB/core)</label>
              <input type="number" className="border rounded p-2" value={vvfGiBPerCore} onChange={e=>setVvfGiBPerCore(parseFloat(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">VCF (TiB/core)</label>
              <input type="number" className="border rounded p-2" value={vcfTiBPerCore} onChange={e=>setVcfTiBPerCore(parseFloat(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">vSAN requerida (TiB)</label>
              <input type="number" className="border rounded p-2" value={vsanRequiredTiB} onChange={e=>setVsanRequiredTiB(parseFloat(e.target.value||"0"))} />
            </div>
          </section>

          {/* Add-ons */}
          <section className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <div className="flex flex-col">
              <label className="text-sm font-semibold">Firewall cores (override opcional)</label>
              <input type="number" className="border rounded p-2" value={firewallCoresOverride} onChange={e=>setFirewallCoresOverride(parseInt(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">Live Recovery — VMs</label>
              <input type="number" className="border rounded p-2" value={liveRecoveryVMs} onChange={e=>setLiveRecoveryVMs(parseInt(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">Live Cyber Recovery — VMs</label>
              <input type="number" className="border rounded p-2" value={liveCyberRecoveryVMs} onChange={e=>setLiveCyberRecoveryVMs(parseInt(e.target.value||"0"))} />
            </div>
            <div className="flex flex-col">
              <label className="text-sm font-semibold">Live Cyber Recovery — Capacidad (TiB)</label>
              <input type="number" className="border rounded p-2" value={liveCyberRecoveryTiB} onChange={e=>setLiveCyberRecoveryTiB(parseFloat(e.target.value||"0"))} />
            </div>
          </section>

          {/* Inventario */}
          <section className="space-y-2">
            <div className="flex items-center justify-between">
              <h2 className="text-xl font-semibold">Inventario de servidores</h2>
              <div className="space-x-2">
                <button onClick={addRow} className="px-3 py-2 rounded bg-gray-100 hover:bg-gray-200">Añadir fila</button>
                <button onClick={resetExample} className="px-3 py-2 rounded bg-gray-100 hover:bg-gray-200">Reset ejemplo</button>
                <button onClick={exportCsv} className="px-3 py-2 rounded bg-gray-900 text-white">Exportar CSV</button>
              </div>
            </div>
            <div className="overflow-auto">
              <table className="w-full border-collapse text-sm">
                <thead>
                  <tr className="bg-gray-50">
                    <th className="p-2 border text-left">Grupo</th>
                    <th className="p-2 border text-right">Servidores</th>
                    <th className="p-2 border text-right">CPUs/Servidor</th>
                    <th className="p-2 border text-right">Cores/CPU (físicos)</th>
                    <th className="p-2 border text-right">CPUs totales</th>
                    <th className="p-2 border text-right">Cores físicos</th>
                    <th className="p-2 border text-right">Cores suscritos/CPU</th>
                    <th className="p-2 border text-right">Cores suscritos</th>
                    <th className="p-2 border text-right">Delta</th>
                  </tr>
                </thead>
                <tbody>
                  {rows.map((r, idx) => (
                    <tr key={r.id}>
                      <td className="p-1 border">
                        <input className="w-full p-1" value={r.name}
                               onChange={e=>setRows(rows.map(rr=>rr.id===r.id?{...rr,name:e.target.value}:rr))} />
                      </td>
                      <td className="p-1 border text-right">
                        <input type="number" className="w-full p-1 text-right" value={r.servers}
                               onChange={e=>setRows(rows.map(rr=>rr.id===r.id?{...rr,servers:parseInt(e.target.value||"0")}:rr))} />
                      </td>
                      <td className="p-1 border text-right">
                        <input type="number" className="w-full p-1 text-right" value={r.cpusPerServer}
                               onChange={e=>setRows(rows.map(rr=>rr.id===r.id?{...rr,cpusPerServer:parseInt(e.target.value||"0")}:rr))} />
                      </td>
                      <td className="p-1 border text-right">
                        <input type="number" className="w-full p-1 text-right" value={r.coresPerCpu}
                               onChange={e=>setRows(rows.map(rr=>rr.id===r.id?{...rr,coresPerCpu:parseInt(e.target.value||"0")}:rr))} />
                      </td>
                      <td className="p-1 border text-right">{calc.details[idx]?.totalCpus ?? 0}</td>
                      <td className="p-1 border text-right">{calc.details[idx]?.physicalCores ?? 0}</td>
                      <td className="p-1 border text-right">{calc.details[idx]?.subscribedPerCpu ?? 0}</td>
                      <td className="p-1 border text-right">{calc.details[idx]?.subscribedCores ?? 0}</td>
                      <td className="p-1 border text-right">{calc.details[idx]?.delta ?? 0}</td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </section>

          {/* Resumen */}
          <section className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div className="border rounded-xl p-4 shadow-sm">
              <h3 className="font-semibold mb-2">Resumen de cómputo</h3>
              <div className="grid grid-cols-2 gap-2 text-sm">
                <div className="text-gray-600">CPUs totales</div>
                <div className="text-right font-semibold">{calc.totalCpus}</div>
                <div className="text-gray-600">Cores físicos totales</div>
                <div className="text-right font-semibold">{calc.totalPhysical}</div>
                <div className="text-gray-600">Cores suscritos totales</div>
                <div className="text-right font-semibold">{calc.totalSubscribed}</div>
                <div className="text-gray-600">Delta (suscrito − físico)</div>
                <div className="text-right font-semibold">{calc.totalDelta}</div>
              </div>
            </div>

            <div className="border rounded-xl p-4 shadow-sm">
              <h3 className="font-semibold mb-2">vSAN incluido vs. requerido (TiB)</h3>
              <div className="grid grid-cols-2 gap-2 text-sm">
                <div className="text-gray-600">Requerido</div>
                <div className="text-right font-semibold">{Number(vsanRequiredTiB) || 0}</div>
                <div className="text-gray-600">Incluido VVF</div>
                <div className="text-right font-semibold">{calc.includedVvfTiB.toFixed(2)}</div>
                <div className="text-gray-600">Incluido VCF</div>
                <div className="text-right font-semibold">{calc.includedVcfTiB.toFixed(2)}</div>
                <div className="text-gray-600">Adicional VVF</div>
                <div className="text-right font-semibold">{calc.addVvfTiB.toFixed(2)}</div>
                <div className="text-gray-600">Adicional VCF</div>
                <div className="text-right font-semibold">{calc.addVcfTiB.toFixed(2)}</div>
              </div>
            </div>
          </section>

          {/* Add-ons Summary */}
          <section className="border rounded-xl p-4 shadow-sm">
            <h3 className="font-semibold mb-2">Add-ons (cantidades a suscribir)</h3>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-2 text-sm">
              <div className="text-gray-600">Firewall cores</div>
              <div className="text-right font-semibold">{calc.firewallCores}</div>
              <div className="text-gray-600">Live Recovery VMs</div>
              <div className="text-right font-semibold">{calc.liveRecoveryVMs}</div>
              <div className="text-gray-600">Live Cyber Recovery VMs</div>
              <div className="text-right font-semibold">{calc.liveCyberRecoveryVMs}</div>
              <div className="text-gray-600">Live Cyber Recovery Capacidad (TiB)</div>
              <div className="text-right font-semibold">{calc.liveCyberRecoveryTiB}</div>
            </div>
          </section>

          <footer className="text-xs text-gray-500">Nota: Este cálculo refleja reglas de suscripción y capacidad incluida. Ajusta inputs para tus escenarios. 1 TiB = 1024 GiB.</footer>
        </div>
      );
    }

    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
  </script>
</body>
</html>
