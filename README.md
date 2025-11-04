# Smart-city
'use client';

import React, { useState, useEffect, useRef, useMemo, useCallback } from 'react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import { Play, Pause, RotateCcw, Car, Wind, AlertTriangle, Zap, Map, Activity, Cloud, Bell, TrendingUp, TrendingDown } from 'lucide-react';

// Types
interface Intersection {
  id: number;
  name: string;
  x: number;
  y: number;
  vehicles: number;
  zone: 'downtown' | 'residential' | 'industrial';
}

interface Alert {
  id: string;
  message: string;
  location: string;
  icon: React.ElementType;
  time: string;
  critical: boolean;
  type: string;
}

interface ChartData {
  time: string;
  traffic: number;
  pm25: number;
  no2: number;
}

export default function SmartCityDashboard() {
  // Reactive State
  const [currentTime, setCurrentTime] = useState('');
  const [simulationRunning, setSimulationRunning] = useState(true);
  const [simulationSpeed, setSimulationSpeed] = useState(1000);
  const [selectedZone, setSelectedZone] = useState<'all' | 'downtown' | 'residential' | 'industrial'>('all');
  const [weather, setWeather] = useState({ temp: 22 });
  
  const [intersections, setIntersections] = useState<Intersection[]>([]);
  const [chartData, setChartData] = useState<ChartData[]>([]);
  const [alerts, setAlerts] = useState<Alert[]>([]);
  const [energy, setEnergy] = useState(0);
  
  const simulationInterval = useRef<NodeJS.Timeout | null>(null);

  // Initialize intersections and data
  useEffect(() => {
    const initialIntersections: Intersection[] = [
      { id: 1, name: 'Central Sq', x: 30, y: 40, vehicles: 45, zone: 'downtown' },
      { id: 2, name: 'Main St', x: 50, y: 30, vehicles: 60, zone: 'downtown' },
      { id: 3, name: 'Park Ave', x: 70, y: 50, vehicles: 25, zone: 'residential' },
      { id: 4, name: 'Factory Rd', x: 40, y: 70, vehicles: 80, zone: 'industrial' },
      { id: 5, name: 'River Dr', x: 60, y: 75, vehicles: 35, zone: 'residential' },
      { id: 6, name: 'Tech Hub', x: 80, y: 30, vehicles: 55, zone: 'downtown' },
    ];
    
    setIntersections(initialIntersections);
    
    const initialData = Array(60).fill(null).map((_, i) => ({
      time: `${59 - i}m ago`,
      traffic: 40 + Math.random() * 20,
      pm25: 35 + Math.random() * 15,
      no2: 25 + Math.random() * 10,
    }));
    setChartData(initialData);
  }, []);

  // Clock and weather
  useEffect(() => {
    const timer = setInterval(() => {
      const now = new Date();
      setCurrentTime(now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' }));
      setWeather(prev => ({
        temp: 20 + Math.round(Math.sin(now.getHours() / 24 * Math.PI) * 5)
      }));
    }, 1000);
    
    return () => clearInterval(timer);
  }, []);

  // Simulation loop
  useEffect(() => {
    if (simulationInterval.current) {
      clearInterval(simulationInterval.current);
    }

    if (simulationRunning) {
      simulationInterval.current = setInterval(() => {
        updateTraffic();
        updatePollution();
        updateEnergy();
        checkAlerts();
      }, simulationSpeed);
    }

    return () => {
      if (simulationInterval.current) {
        clearInterval(simulationInterval.current);
      }
    };
  }, [simulationRunning, simulationSpeed]);

  // Computed values using useMemo
  const filteredIntersections = useMemo(() => {
    if (selectedZone === 'all') return intersections;
    return intersections.filter(i => i.zone === selectedZone);
  }, [intersections, selectedZone]);

  const avgTraffic = useMemo(() => {
    if (filteredIntersections.length === 0) return 0;
    const total = filteredIntersections.reduce((sum, i) => sum + i.vehicles, 0);
    return Math.round(total / filteredIntersections.length);
  }, [filteredIntersections]);

  const trafficTrend = useMemo(() => {
    if (chartData.length < 2) return 0;
    const last = chartData[chartData.length - 1].traffic;
    const prev = chartData[chartData.length - 2].traffic;
    return Math.round(((last - prev) / prev) * 100);
  }, [chartData]);

  const aqi = useMemo(() => {
    if (chartData.length === 0) return 50;
    const latest = chartData[chartData.length - 1];
    return Math.round((latest.pm25 * 1.5 + latest.no2) / 2);
  }, [chartData]);

  const aqiTrend = useMemo(() => {
    if (chartData.length < 2) return 0;
    const last = chartData[chartData.length - 1];
    const prev = chartData[chartData.length - 2];
    const lastAQI = (last.pm25 * 1.5 + last.no2) / 2;
    const prevAQI = (prev.pm25 * 1.5 + prev.no2) / 2;
    return Math.round(lastAQI - prevAQI);
  }, [chartData]);

  const trafficLevel = useMemo(() => {
    if (avgTraffic < 30) return { text: 'Low', color: 'text-green-400', bg: 'bg-green-500/20' };
    if (avgTraffic < 70) return { text: 'Moderate', color: 'text-amber-400', bg: 'bg-amber-500/20' };
    return { text: 'High', color: 'text-red-400', bg: 'bg-red-500/20' };
  }, [avgTraffic]);

  const aqiLevel = useMemo(() => {
    if (aqi <= 50) return { text: 'Good', color: 'text-green-400', bg: 'bg-green-500/20' };
    if (aqi <= 100) return { text: 'Moderate', color: 'text-amber-400', bg: 'bg-amber-500/20' };
    return { text: 'Poor', color: 'text-red-400', bg: 'bg-red-500/20' };
  }, [aqi]);

  const criticalAlerts = useMemo(() => {
    return alerts.filter(a => a.critical).length;
  }, [alerts]);

  // Update functions
  const updateTraffic = useCallback(() => {
    setIntersections(prev => prev.map(intersection => {
      const base = intersection.zone === 'downtown' ? 60 : intersection.zone === 'industrial' ? 70 : 30;
      const variation = Math.sin(Date.now() / 10000 + intersection.id) * 20;
      const rushHour = new Date().getHours() >= 7 && new Date().getHours() <= 9 ? 1.5 : 1;
      const newVehicles = Math.max(0, Math.min(100, Math.round(base + variation * rushHour + Math.random() * 10)));
      
      return { ...intersection, vehicles: newVehicles };
    }));

    setChartData(prev => {
      const newData = [...prev.slice(1)];
      newData.push({
        time: 'now',
        traffic: avgTraffic,
        pm25: prev[prev.length - 1].pm25,
        no2: prev[prev.length - 1].no2,
      });
      return newData;
    });
  }, [avgTraffic]);

  const updatePollution = useCallback(() => {
    const trafficFactor = avgTraffic / 50;
    const weatherFactor = weather.temp > 28 ? 1.3 : 1;
    const basePM25 = 30 + trafficFactor * 40 + Math.random() * 15;
    const baseNO2 = 20 + trafficFactor * 30 + Math.random() * 10;
    
    setChartData(prev => {
      const newData = [...prev.slice(1)];
      newData.push({
        time: 'now',
        traffic: prev[prev.length - 1].traffic,
        pm25: Math.round(basePM25 * weatherFactor),
        no2: Math.round(baseNO2 * weatherFactor),
      });
      return newData;
    });
  }, [avgTraffic, weather.temp]);

  const updateEnergy = useCallback(() => {
    const base = 50;
    const trafficLoad = avgTraffic * 0.5;
    setEnergy(Math.round(base + trafficLoad + Math.random() * 10));
  }, [avgTraffic]);

  const checkAlerts = useCallback(() => {
    const now = new Date().toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' });
    const highTraffic = filteredIntersections.filter(i => i.vehicles > 85);
    const poorAQI = aqi > 120;

    const newAlerts: Alert[] = [];

    highTraffic.forEach(int => {
      if (!alerts.some(a => a.location === int.name && a.type === 'congestion')) {
        newAlerts.push({
          id: Date.now() + Math.random().toString(),
          message: `Severe congestion at ${int.name}`,
          location: int.name,
          icon: AlertTriangle,
          time: now,
          critical: true,
          type: 'congestion'
        });
      }
    });

    if (poorAQI && !alerts.some(a => a.type === 'aqi')) {
      newAlerts.push({
        id: Date.now().toString(),
        message: `Air quality alert: AQI ${aqi} (Poor)`,
        location: 'Citywide',
        icon: Cloud,
        time: now,
        critical: true,
        type: 'aqi'
      });
    }

    if (newAlerts.length > 0) {
      setAlerts(prev => [...prev, ...newAlerts]);
    }

    // Expire old alerts (5 minutes)
    setAlerts(prev => prev.filter(a => Date.now() - parseInt(a.id) < 300000));
  }, [filteredIntersections, aqi, alerts]);

  // Control functions
  const toggleSimulation = () => setSimulationRunning(prev => !prev);
  
  const reset = () => {
    setIntersections(prev => prev.map(i => ({ ...i, vehicles: 40 + Math.random() * 20 })));
    setAlerts([]);
    setEnergy(65);
    setChartData(prev => prev.map(d => ({ ...d, traffic: 40 + Math.random() * 20 })));
  };

  const selectIntersection = (intersection: Intersection) => {
    const status = intersection.vehicles < 30 ? 'Smooth' : intersection.vehicles < 70 ? 'Moderate' : 'Congested';
    alert(`Intersection: ${intersection.name}\nVehicles: ${intersection.vehicles}\nZone: ${intersection.zone}\nStatus: ${status}`);
  };

  return (
    <>
      <div className="min-h-screen bg-slate-900 text-slate-100 p-4">
        <div className="max-w-7xl mx-auto">
          {/* Header */}
          <div className="bg-slate-800 rounded-xl p-6 mb-6 border border-slate-700">
            <div className="flex justify-between items-center mb-4">
              <h1 className="text-2xl font-bold flex items-center gap-2">
                <Map className="w-6 h-6" />
                Smart City Traffic & Pollution Monitor
              </h1>
              <div className="text-sm">
                <span>{currentTime}</span> | 
                <span className="ml-2">Temp: {weather.temp}°C</span>
              </div>
            </div>
            
            <div className="flex flex-wrap gap-3">
              <button
                onClick={toggleSimulation}
                className={`flex items-center gap-2 px-4 py-2 rounded-lg font-medium transition-colors ${
                  simulationRunning ? 'bg-green-600 hover:bg-green-700' : 'bg-blue-600 hover:bg-blue-700'
                }`}
              >
                {simulationRunning ? <Pause className="w-4 h-4" /> : <Play className="w-4 h-4" />}
                {simulationRunning ? 'Pause' : 'Play'}
              </button>
              
              <button
                onClick={reset}
                className="flex items-center gap-2 px-4 py-2 bg-slate-700 hover:bg-slate-600 rounded-lg font-medium transition-colors"
              >
                <RotateCcw className="w-4 h-4" />
                Reset
              </button>
              
              <div className="flex items-center gap-2">
                <label className="text-sm">Speed:</label>
                <input
                  type="range"
                  min="100"
                  max="3000"
                  step="100"
                  value={simulationSpeed}
                  onChange={(e) => setSimulationSpeed(Number(e.target.value))}
                  className="w-32"
                />
                <span className="text-sm w-16">{simulationSpeed}ms</span>
              </div>
              
              <select
                value={selectedZone}
                onChange={(e) => setSelectedZone(e.target.value as any)}
                className="bg-slate-700 text-slate-100 px-3 py-2 rounded-lg border border-slate-600"
              >
                <option value="all">All Zones</option>
                <option value="downtown">Downtown</option>
                <option value="residential">Residential</option>
                <option value="industrial">Industrial</option>
              </select>
            </div>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-6">
            {/* KPI Cards */}
            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex justify-between items-start mb-3">
                <div className="flex items-center gap-2">
                  <Car className="w-5 h-5 text-blue-400" />
                  <span className="text-sm font-medium">Avg Traffic Density</span>
                </div>
                <span className={`px-2 py-1 rounded text-xs font-semibold ${trafficLevel.bg} ${trafficLevel.color}`}>
                  {trafficLevel.text}
                </span>
              </div>
              <div className="text-3xl font-bold mb-1">{avgTraffic}</div>
              <div className={`flex items-center gap-1 text-sm ${trafficTrend >= 0 ? 'text-green-400' : 'text-red-400'}`}>
                {trafficTrend >= 0 ? <TrendingUp className="w-4 h-4" /> : <TrendingDown className="w-4 h-4" />}
                {Math.abs(trafficTrend)}% vs last hour
              </div>
            </div>

            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex justify-between items-start mb-3">
                <div className="flex items-center gap-2">
                  <Wind className="w-5 h-5 text-purple-400" />
                  <span className="text-sm font-medium">Air Quality Index (AQI)</span>
                </div>
                <span className={`px-2 py-1 rounded text-xs font-semibold ${aqiLevel.bg} ${aqiLevel.color}`}>
                  {aqiLevel.text}
                </span>
              </div>
              <div className="text-3xl font-bold mb-1">{aqi}</div>
              <div className={`flex items-center gap-1 text-sm ${aqiTrend >= 0 ? 'text-red-400' : 'text-green-400'}`}>
                {aqiTrend >= 0 ? <TrendingUp className="w-4 h-4" /> : <TrendingDown className="w-4 h-4" />}
                {Math.abs(aqiTrend)} vs last hour
              </div>
            </div>

            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex items-center gap-2 mb-3">
                <AlertTriangle className="w-5 h-5 text-orange-400" />
                <span className="text-sm font-medium">Active Alerts</span>
              </div>
              <div className="text-2xl font-bold mb-1">{alerts.length}</div>
              <div className="text-sm text-slate-400">{criticalAlerts} critical</div>
            </div>

            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex items-center gap-2 mb-3">
                <Zap className="w-5 h-5 text-yellow-400" />
                <span className="text-sm font-medium">Energy Consumption</span>
              </div>
              <div className="text-2xl font-bold mb-1">{energy} kWh</div>
              <div className="text-sm text-slate-400">Traffic lights & sensors</div>
            </div>
          </div>

          {/* City Map */}
          <div className="bg-slate-800 rounded-xl p-6 mb-6 border border-slate-700">
            <div className="flex justify-between items-center mb-4">
              <div className="flex items-center gap-2">
                <Map className="w-5 h-5" />
                <h2 className="text-lg font-semibold">Live City Map</h2>
              </div>
              <div className="flex gap-4 text-sm">
                <div className="flex items-center gap-2">
                  <div className="w-3 h-3 rounded-full bg-green-500"></div>
                  <span>Low Traffic</span>
                </div>
                <div className="flex items-center gap-2">
                  <div className="w-3 h-3 rounded-full bg-amber-500"></div>
                  <span>Moderate</span>
                </div>
                <div className="w-3 h-3 rounded-full bg-red-500"></div>
                <span>High</span>
              </div>
            </div>
            
            <div className="relative h-96 bg-slate-900 rounded-lg overflow-hidden border border-slate-700">
              {filteredIntersections.map(intersection => {
                const color = intersection.vehicles < 30 ? '#10b981' : intersection.vehicles < 70 ? '#f59e0b' : '#ef4444';
                return (
                  <div
                    key={intersection.id}
                    className="absolute w-4 h-4 rounded-full cursor-pointer transition-all hover:scale-150 hover:z-10"
                    style={{
                      left: `${intersection.x}%`,
                      top: `${intersection.y}%`,
                      backgroundColor: color,
                      transform: 'translate(-50%, -50%)',
                      boxShadow: `0 0 12px ${color}`
                    }}
                    onClick={() => selectIntersection(intersection)}
                  >
                    <div className="absolute -top-8 left-1/2 -translate-x-1/2 bg-slate-900 px-2 py-1 rounded text-xs whitespace-nowrap opacity-0 transition-opacity hover:opacity-100">
                      {intersection.name}
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
            {/* Traffic Chart */}
            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex items-center gap-2 mb-4">
                <Activity className="w-5 h-5" />
                <h2 className="text-lg font-semibold">Traffic Flow (Last 60 min)</h2>
              </div>
              <div className="h-64">
                <ResponsiveContainer width="100%" height="100%">
                  <LineChart data={chartData}>
                    <CartesianGrid strokeDasharray="3 3" stroke="#374151" />
                    <XAxis dataKey="time" stroke="#94a3b8" fontSize={12} />
                    <YAxis stroke="#94a3b8" fontSize={12} />
                    <Tooltip 
                      contentStyle={{ backgroundColor: '#1f2937', border: '1px solid #374151' }}
                      labelStyle={{ color: '#e5e7eb' }}
                    />
                    <Line 
                      type="monotone" 
                      dataKey="traffic" 
                      stroke="#3b82f6" 
                      strokeWidth={2}
                      dot={false}
                      name="Vehicles/min"
                    />
                  </LineChart>
                </ResponsiveContainer>
              </div>
            </div>

            {/* Pollution Chart */}
            <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
              <div className="flex items-center gap-2 mb-4">
                <Cloud className="w-5 h-5" />
                <h2 className="text-lg font-semibold">Pollution Levels</h2>
              </div>
              <div className="h-64">
                <ResponsiveContainer width="100%" height="100%">
                  <LineChart data={chartData}>
                    <CartesianGrid strokeDasharray="3 3" stroke="#374151" />
                    <XAxis dataKey="time" stroke="#94a3b8" fontSize={12} />
                    <YAxis stroke="#94a3b8" fontSize={12} />
                    <Tooltip 
                      contentStyle={{ backgroundColor: '#1f2937', border: '1px solid #374151' }}
                      labelStyle={{ color: '#e5e7eb' }}
                    />
                    <Legend />
                    <Line 
                      type="monotone" 
                      dataKey="pm25" 
                      stroke="#8b5cf6" 
                      strokeWidth={2}
                      dot={false}
                      name="PM2.5 (μg/m³)"
                    />
                    <Line 
                      type="monotone" 
                      dataKey="no2" 
                      stroke="#ec4899" 
                      strokeWidth={2}
                      dot={false}
                      name="NO₂ (ppb)"
                    />
                  </LineChart>
                </ResponsiveContainer>
              </div>
            </div>
          </div>

          {/* Alerts Panel */}
          <div className="bg-slate-800 rounded-xl p-6 border border-slate-700">
            <div className="flex items-center gap-2 mb-4">
              <Bell className="w-5 h-5" />
              <h2 className="text-lg font-semibold">Real-time Alerts</h2>
            </div>
            
            <div className="max-h-80 overflow-y-auto">
              {alerts.length === 0 ? (
                <div className="text-center py-8 text-slate-400">
                  No active alerts
                </div>
              ) : (
                <div className="space-y-2">
                  {[...alerts].reverse().map(alert => {
                    const Icon = alert.icon;
                    return (
                      <div key={alert.id} className={`flex items-center gap-3 p-3 rounded-lg border ${alert.critical ? 'border-red-500/50 bg-red-500/10' : 'border-slate-700'}`}>
                        <Icon className={`w-5 h-5 ${alert.critical ? 'text-red-400' : 'text-slate-400'}`} />
                        <div className="flex-1">
                          <div className="font-medium">{alert.message}</div>
                          <div className="text-xs text-slate-400">{alert.location}</div>
                        </div>
                        <div className="text-xs text-slate-400">{alert.time}</div>
                      </div>
                    );
                  })}
                </div>
              )}
            </div>
          </div>
        </div>
      </div>
    </>
  );
}
