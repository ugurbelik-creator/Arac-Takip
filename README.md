import { useState, useCallback } from 'react';
import VehicleList from '@/components/VehicleList';
import VehicleCheckout from '@/components/VehicleCheckout';
import ActiveVehicles from '@/components/ActiveVehicles';
import AssignmentHistory from '@/components/AssignmentHistory';
import type { Vehicle, ActiveVehicle, HistoryEntry } from '@/types';
import { toast } from 'sonner';

function generateId() {
  return Math.random().toString(36).substring(2, 11);
}

function formatDate() {
  return new Date().toLocaleString('tr-TR', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  });
}

export default function Index() {
  const [vehicles, setVehicles] = useState<Vehicle[]>([]);
  const [activeVehicles, setActiveVehicles] = useState<ActiveVehicle[]>([]);
  const [history, setHistory] = useState<HistoryEntry[]>([]);

  const addVehicle = useCallback(
    (plate: string, contractor: string, brand: string, model: string) => {
      setVehicles((prev) => [
        ...prev,
        {
          id: generateId(),
          plate,
          contractor,
          brand,
          model,
          addedAt: formatDate(),
        },
      ]);
    },
    []
  );

  const removeVehicle = useCallback((id: string) => {
    setVehicles((prev) => prev.filter((v) => v.id !== id));
  }, []);

  const importVehicles = useCallback(
    (rows: { plate: string; contractor: string; brand: string; model: string }[]) => {
      setVehicles((prev) => {
        const existingPlates = new Set(prev.map((v) => v.plate));
        const newVehicles = rows
          .filter((r) => !existingPlates.has(r.plate))
          .map((r) => ({
            id: generateId(),
            plate: r.plate,
            contractor: r.contractor,
            brand: r.brand,
            model: r.model,
            addedAt: formatDate(),
          }));
        return [...prev, ...newVehicles];
      });
    },
    []
  );

  const handleUpdateAddedAt = useCallback(
    (id: string, newAddedAt: string) => {
      setVehicles((prev) =>
        prev.map((v) => (v.id === id ? { ...v, addedAt: newAddedAt } : v))
      );
    },
    []
  );

  const handleCheckout = useCallback(
    (plate: string, user: string, unit: string, description: string) => {
      // Aynı plaka zaten kullanımda mı kontrol et
      if (activeVehicles.some((v) => v.plate === plate)) {
        toast.error(`${plate} plakası zaten kullanımda.`);
        return;
      }

      // Sol listedeyse, orijinal verileri al ve listeden kaldır
      const vehicleInList = vehicles.find((v) => v.plate === plate);
      if (vehicleInList) {
        setVehicles((prev) => prev.filter((v) => v.plate !== plate));
      }

      // Kullanılan araçlar listesine ekle (orijinal verileri koru)
      const now = formatDate();
      const newActive: ActiveVehicle = {
        id: generateId(),
        plate,
        user,
        unit,
        description,
        checkoutTime: now,
        contractor: vehicleInList?.contractor || '',
        brand: vehicleInList?.brand || '',
        model: vehicleInList?.model || '',
        addedAt: vehicleInList?.addedAt || now,
      };
      setActiveVehicles((prev) => [...prev, newActive]);

      toast.success(`${plate} araç çıkışı yapıldı.`);
    },
    [vehicles, activeVehicles]
  );

  const handleReturn = useCallback(
    (activeId: string) => {
      const vehicle = activeVehicles.find((v) => v.id === activeId);
      if (!vehicle) return;

      // Kullanılan araçlardan kaldır
      setActiveVehicles((prev) => prev.filter((v) => v.id !== activeId));

      // Kısa süreli listeye orijinal verileriyle geri ekle
      setVehicles((prev) => [
        ...prev,
        {
          id: generateId(),
          plate: vehicle.plate,
          contractor: vehicle.contractor,
          brand: vehicle.brand,
          model: vehicle.model,
          addedAt: vehicle.addedAt,
        },
      ]);

      // Geçmişe yaz
      const now = formatDate();
      const historyEntry: HistoryEntry = {
        id: generateId(),
        plate: vehicle.plate,
        user: vehicle.user,
        unit: vehicle.unit,
        description: vehicle.description,
        action: 'returned',
        timestamp: now,
        checkoutTime: vehicle.checkoutTime,
        returnTime: now,
      };
      setHistory((prev) => [...prev, historyEntry]);

      toast.success(`${vehicle.plate} araç geri döndü ve listeye eklendi.`);
    },
    [activeVehicles]
  );

  const handleRefund = useCallback(
    (activeId: string) => {
      const vehicle = activeVehicles.find((v) => v.id === activeId);
      if (!vehicle) return;

      // Kullanılan araçlardan kaldır (listeye geri eklemeden)
      setActiveVehicles((prev) => prev.filter((v) => v.id !== activeId));

      // Geçmişe yaz
      const now = formatDate();
      const historyEntry: HistoryEntry = {
        id: generateId(),
        plate: vehicle.plate,
        user: vehicle.user,
        unit: vehicle.unit,
        description: vehicle.description,
        action: 'refunded',
        timestamp: now,
        checkoutTime: vehicle.checkoutTime,
        returnTime: now,
      };
      setHistory((prev) => [...prev, historyEntry]);

      toast.info(`${vehicle.plate} araç iade edildi ve listeden çıkarıldı.`);
    },
    [activeVehicles]
  );

  const handleUpdateTimestamp = useCallback(
    (id: string, newTimestamp: string) => {
      setHistory((prev) =>
        prev.map((entry) =>
          entry.id === id ? { ...entry, timestamp: newTimestamp } : entry
        )
      );
    },
    []
  );

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-blue-50">
      {/* Header */}
      <header className="bg-gradient-to-r from-slate-800 to-slate-900 text-white shadow-xl">
        <div className="max-w-[1920px] mx-auto px-4 py-4 flex items-center gap-3">
          <div className="bg-blue-500 p-2 rounded-lg">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              className="h-6 w-6"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              strokeWidth={2}
            >
              <path
                strokeLinecap="round"
                strokeLinejoin="round"
                d="M9 17a2 2 0 11-4 0 2 2 0 014 0zM19 17a2 2 0 11-4 0 2 2 0 014 0z"
              />
              <path
                strokeLinecap="round"
                strokeLinejoin="round"
                d="M13 16V6a1 1 0 00-1-1H4a1 1 0 00-1 1v10M17 16V8a1 1 0 00-.76-.97L13 6"
              />
            </svg>
          </div>
          <div>
            <h1 className="text-xl font-bold tracking-tight">
              Araç Takip Sistemi
            </h1>
            <p className="text-xs text-slate-300">
              Kısa Süreli Kiralık Araç Yönetimi
            </p>
          </div>
        </div>
      </header>

      {/* Main Content */}
      <main className="max-w-[1920px] mx-auto p-4 h-[calc(100vh-72px)]">
        <div className="flex gap-4 h-full">
          {/* Sol Panel - %40 */}
          <div className="w-[40%] min-w-[320px]">
            <VehicleList
              vehicles={vehicles}
              onAddVehicle={addVehicle}
              onRemoveVehicle={removeVehicle}
              onImportVehicles={importVehicles}
              onUpdateAddedAt={handleUpdateAddedAt}
            />
          </div>

          {/* Sağ Panel - %60, 3'e bölünmüş */}
          <div className="w-[60%] flex flex-col gap-4">
            {/* Sağ Üst - Araç Çıkış */}
            <div className="shrink-0">
              <VehicleCheckout onCheckout={handleCheckout} />
            </div>

            {/* Sağ Orta - Kullanılan Araçlar */}
            <div className="flex-1 min-h-0">
              <ActiveVehicles
                vehicles={activeVehicles}
                onReturn={handleReturn}
                onRefund={handleRefund}
              />
            </div>

            {/* Sağ Alt - Görevlendirme Geçmişi */}
            <div className="flex-1 min-h-0">
              <AssignmentHistory
                history={history}
                onUpdateTimestamp={handleUpdateTimestamp}
              />
            </div>
          </div>
        </div>
      </main>
    </div>
  );
}
