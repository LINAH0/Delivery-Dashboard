import React, { useMemo, useState } from "react";
import Papa from "papaparse";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { TrendingUp, Truck, Clock3, CheckCircle2 } from "lucide-react";
import { Line, Bar } from "react-chartjs-2";
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
} from "chart.js";

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, BarElement, Title, Tooltip, Legend);

// Expected columns
const COLS = {
  PO: "PO_No",
  SEASON: "Season",
  STYLE: "Style",
  SKU: "SKU",
  PRODUCT: "Product_Type",
  VENDOR: "Vendor",
  FACTORY: "Factory",
  ORIGIN: "Country_Origin",
  SHIP_METHOD: "Shipping_Method",
  WAYBILL: "Waybill_No",
  QTY: "Order_Qty",
  UNIT: "Unit",
  CURR: "Currency",
  RDD: "RDD(N)",
  ETD: "ETD",
  ETA: "ETA",
  ACTUAL: "Actual_Arrival",
  STATUS: "Status",
  DELAY: "Delay_Days",
  ONTIME: "On_Time(YesNo)",
  T: "T_Arrival_Possible",
  U: "U_Main_Supply_or_Rework(O/X)",
  V: "V_Substitute_Material_Memo",
  W: "W_CN_Direct_Ship_Date",
  X: "X_CN_Direct_Waybill",
  SMS: "SMS_Arrival_Date",
  NOTES: "Notes",
};

function parseCSV(file, onDone) {
  Papa.parse(file, {
    header: true,
    skipEmptyLines: true,
    complete: (results) => onDone(results.data),
  });
}

function kpiCompute(rows) {
  const arrived = rows.filter((r) => (r[COLS.STATUS] || "").toLowerCase() === "arrived");
  const on = arrived.filter((r) => (r[COLS.ONTIME] || "").toLowerCase() === "yes").length;
  const onRate = arrived.length ? +(on * 100 / arrived.length).toFixed(1) : 0;
  const avgDelay = arrived.length
    ? +(arrived.reduce((s, r) => s + Number(r[COLS.DELAY] || 0), 0) / arrived.length).toFixed(2)
    : 0;
  const inTransit = rows.filter((r) => (r[COLS.STATUS] || "").toLowerCase() === "in-transit").length;
  return { onRate, avgDelay, total: rows.length, arrived: arrived.length, inTransit };
}

export default function DeliveryDashboard() {
  const [rows, setRows] = useState([]);
  const [vendor, setVendor] = useState("ALL");
  const [origin, setOrigin] = useState("ALL");
  const [season, setSeason] = useState("ALL");
  const [q, setQ] = useState("");

  const filtered = useMemo(() => {
    return rows.filter((r) => {
      const v = vendor === "ALL" || r[COLS.VENDOR] === vendor;
      const o = origin === "ALL" || r[COLS.ORIGIN] === origin;
      const s = season === "ALL" || r[COLS.SEASON] === season;
      const text = `${r[COLS.PO]} ${r[COLS.SKU]} ${r[COLS.STYLE]} ${r[COLS.WAYBILL]} ${r[COLS.NOTES]}`.toLowerCase();
      const t = !q || text.includes(q.toLowerCase());
      return v && o && s && t;
    });
  }, [rows, vendor, origin, season, q]);

  const k = useMemo(() => kpiCompute(filtered), [filtered]);

  const vendorStats = useMemo(() => {
    const map = new Map();
    for (const r of filtered) {
      const v = r[COLS.VENDOR];
      if (!v) continue;
      const key = v;
      const arr = (r[COLS.STATUS] || "").toLowerCase() === "arrived";
      const on = (r[COLS.ONTIME] || "").toLowerCase() === "yes";
      if (!map.has(key)) map.set(key, { total: 0, arrived: 0, on: 0 });
      const obj = map.get(key);
      obj.total += 1;
      if (arr) {
        obj.arrived += 1;
        if (on) obj.on += 1;
      }
    }
    const labels = Array.from(map.keys());
    const data = labels.map((l) => {
      const { arrived, on } = map.get(l);
      return arrived ? +(on * 100 / arrived).toFixed(1) : 0;
    });
    return { labels, data };
  }, [filtered]);

  const byEtaLine = useMemo(() => {
    // On-time rate by ETA week
    const bucket = new Map();
    for (const r of filtered) {
      const eta = r[COLS.ETA];
      if (!eta) continue;
      const wk = new Date(eta).toISOString().slice(0, 10);
      if (!bucket.has(wk)) bucket.set(wk, { arrived: 0, on: 0 });
      const b = bucket.get(wk);
      if ((r[COLS.STATUS] || "").toLowerCase() === "arrived") {
        b.arrived += 1;
        if ((r[COLS.ONTIME] || "").toLowerCase() === "yes") b.on += 1;
      }
    }
    const labels = Array.from(bucket.keys()).sort();
    const data = labels.map((d) => {
      const b = bucket.get(d);
      return b.arrived ? +(b.on * 100 / b.arrived).toFixed(1) : 0;
    });
    return { labels, data };
  }, [filtered]);

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-7xl mx-auto space-y-6">
        <div className="flex items-center justify-between">
          <h1 className="text-2xl font-bold">Delivery Dashboard</h1>
          <div className="flex items-center gap-3">
            <Input type="file" accept=".csv" onChange={(e) => {
              const f = e.target.files?.[0];
              if (f) parseCSV(f, setRows);
            }} />
            <Button onClick={() => setRows([])}>Reset</Button>
          </div>
        </div>

        {/* Filters */}
        <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
          <div>
            <Label>Vendor</Label>
            <Select value={vendor} onValueChange={setVendor}>
              <SelectTrigger className="w-full"><SelectValue placeholder="Vendor" /></SelectTrigger>
              <SelectContent>
                <SelectItem value="ALL">ALL</SelectItem>
                {Array.from(new Set(rows.map(r => r[COLS.VENDOR]).filter(Boolean))).map(v => (
                  <SelectItem key={v} value={v}>{v}</SelectItem>
                ))}
              </SelectContent>
            </Select>
          </div>
          <div>
            <Label>Origin</Label>
            <Select value={origin} onValueChange={setOrigin}>
              <SelectTrigger className="w-full"><SelectValue placeholder="Origin" /></SelectTrigger>
              <SelectContent>
                <SelectItem value="ALL">ALL</SelectItem>
                {Array.from(new Set(rows.map(r => r[COLS.ORIGIN]).filter(Boolean))).map(o => (
                  <SelectItem key={o} value={o}>{o}</SelectItem>
                ))}
              </SelectContent>
            </Select>
          </div>
          <div>
            <Label>Season</Label>
            <Select value={season} onValueChange={setSeason}>
              <SelectTrigger className="w-full"><SelectValue placeholder="Season" /></SelectTrigger>
              <SelectContent>
                <SelectItem value="ALL">ALL</SelectItem>
                {Array.from(new Set(rows.map(r => r[COLS.SEASON]).filter(Boolean))).map(s => (
                  <SelectItem key={s} value={s}>{s}</SelectItem>
                ))}
              </SelectContent>
            </Select>
          </div>
          <div>
            <Label>Search</Label>
            <Input placeholder="PO/SKU/Waybill/Notes" value={q} onChange={(e) => setQ(e.target.value)} />
          </div>
        </div>

        {/* KPI Cards */}
        <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
          <Card className="rounded-2xl shadow-sm"><CardContent className="p-4"><div className="text-sm text-gray-500">On-Time Rate</div><div className="text-2xl font-semibold flex items-center gap-2"><CheckCircle2 className="w-5 h-5" />{k.onRate}%</div></CardContent></Card>
          <Card className="rounded-2xl shadow-sm"><CardContent className="p-4"><div className="text-sm text-gray-500">Avg Delay (days)</div><div className="text-2xl font-semibold flex items-center gap-2"><Clock3 className="w-5 h-5" />{k.avgDelay}</div></CardContent></Card>
          <Card className="rounded-2xl shadow-sm"><CardContent className="p-4"><div className="text-sm text-gray-500">Total POs</div><div className="text-2xl font-semibold flex items-center gap-2"><TrendingUp className="w-5 h-5" />{k.total}</div></CardContent></Card>
          <Card className="rounded-2xl shadow-sm"><CardContent className="p-4"><div className="text-sm text-gray-500">In-Transit</div><div className="text-2xl font-semibold flex items-center gap-2"><Truck className="w-5 h-5" />{k.inTransit}</div></CardContent></Card>
        </div>

        {/* Charts */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <Card className="rounded-2xl shadow-sm">
            <CardContent className="p-4">
              <div className="text-sm font-medium mb-2">On-Time Rate by Vendor</div>
              <Bar data={{ labels: vendorStats.labels, datasets: [{ label: "On-Time %", data: vendorStats.data }] }} options={{ responsive: true, plugins: { legend: { display: true } }, scales: { y: { beginAtZero: true, max: 100 } } }} />
            </CardContent>
          </Card>
          <Card className="rounded-2xl shadow-sm">
            <CardContent className="p-4">
              <div className="text-sm font-medium mb-2">On-Time Rate by ETA (daily)</div>
              <Line data={{ labels: byEtaLine.labels, datasets: [{ label: "On-Time %", data: byEtaLine.data }] }} options={{ responsive: true, plugins: { legend: { display: true } }, scales: { y: { beginAtZero: true, max: 100 } } }} />
            </CardContent>
          </Card>
        </div>

        {/* Table */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-4 overflow-auto">
            <table className="min-w-full text-sm">
              <thead className="bg-gray-100 sticky top-0">
                <tr>
                  {Object.values(COLS).map((c) => (
                    <th key={c} className="text-left p-2 whitespace-nowrap">{c}</th>
                  ))}
                </tr>
              </thead>
              <tbody>
                {filtered.map((r, i) => (
                  <tr key={i} className="border-b">
                    {Object.values(COLS).map((c) => (
                      <td key={c} className="p-2 whitespace-nowrap">{r[c]}</td>
                    ))}
                  </tr>
                ))}
              </tbody>
            </table>
          </CardContent>
        </Card>

        <div className="text-xs text-gray-500">* CSV 업로드는 제공된 템플릿 컬럼명을 그대로 사용하세요. RDD(N), ETD, ETA, Actual_Arrival는 ISO 날짜(YYYY-MM-DD) 권장.</div>
      </div>
    </div>
  );
}
