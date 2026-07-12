# TransitOps
Smart Transport Operations Platform for Odoo Hackathon 2026


/app/frontend/src/App.js --file-text "import React from \"react\";
import \"./App.css\";
import { BrowserRouter, Routes, Route, Navigate } from \"react-router-dom\";
import { AuthProvider, useAuth } from \"./lib/auth\";
import { Toaster } from \"sonner\";
import Layout from \"./components/Layout\";
import Login from \"./pages/Login\";
import Dashboard from \"./pages/Dashboard\";
import Dispatch from \"./pages/Dispatch\";
import Emergencies from \"./pages/Emergencies\";
import Ambulances from \"./pages/Ambulances\";
import Drivers from \"./pages/Drivers\";
import Hospitals from \"./pages/Hospitals\";
import Maintenance from \"./pages/Maintenance\";
import FuelExpenses from \"./pages/FuelExpenses\";
import Risk from \"./pages/Risk\";
import LiveMap from \"./pages/LiveMap\";
import Reports from \"./pages/Reports\";

function Protected({ children }) {
  const { user, ready } = useAuth();
  if (!ready) return <div className=\"min-h-screen flex items-center justify-center text-white/40 text-sm\">Loading...</div>;
  if (!user) return <Navigate to=\"/login\" replace />;
  return <Layout>{children}</Layout>;
}

export default function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Toaster theme=\"dark\" position=\"top-right\" richColors />
        <Routes>
          <Route path=\"/login\" element={<Login />} />
          <Route path=\"/\" element={<Protected><Dashboard /></Protected>} />
          <Route path=\"/dispatch\" element={<Protected><Dispatch /></Protected>} />
          <Route path=\"/emergencies\" element={<Protected><Emergencies /></Protected>} />
          <Route path=\"/ambulances\" element={<Protected><Ambulances /></Protected>} />
          <Route path=\"/drivers\" element={<Protected><Drivers /></Protected>} />
          <Route path=\"/hospitals\" element={<Protected><Hospitals /></Protected>} />
          <Route path=\"/maintenance\" element={<Protected><Maintenance /></Protected>} />
          <Route path=\"/fuel\" element={<Protected><FuelExpenses /></Protected>} />
          <Route path=\"/risk\" element={<Protected><Risk /></Protected>} />
          <Route path=\"/map\" element={<Protected><LiveMap /></Protected>} />
          <Route path=\"/reports\" element={<Protected><Reports /></Protected>} />
          <Route path=\"*\" element={<Navigate to=\"/\" replace />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
"
Observation: Overwrite successful: /app/frontend/src/App.js
