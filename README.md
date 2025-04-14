# OB

"use client";
import React from "react";

function MainComponent() {
  const [formData, setFormData] = useState({
    styleNumber: "",
    brandName: "",
    orderQuantity: "",
    garmentType: "",
  });

  const [operations, setOperations] = useState([]);
  const [garmentTypes, setGarmentTypes] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [selectedSection, setSelectedSection] = useState("");
  const [showAddGarmentType, setShowAddGarmentType] = useState(false);
  const [showAddMachineType, setShowAddMachineType] = useState(false);
  const [newGarmentType, setNewGarmentType] = useState("");
  const [newMachineType, setNewMachineType] = useState({
    name: "",
    category: "",
  });
  const [addingGarmentType, setAddingGarmentType] = useState(false);
  const [addingMachineType, setAddingMachineType] = useState(false);

  const sections = [
    { id: "cutting", name: "Cutting", maxOps: 50 },
    { id: "sewing", name: "Sewing", maxOps: 50 },
    { id: "finishing", name: "Finishing", maxOps: 50 },
    { id: "packing", name: "Packing", maxOps: 50 },
  ];

  const machineTypes = [
    "SNLS",
    "DNLS",
    "TNLS",
    "4TOL",
    "5TOL",
    "3TOL",
    "SNCS",
    "DNCS",
    "TNCS",
    "2TFL",
    "3TFL",
    "2NCS",
    "3NCS",
    "4NCS",
    "ABH",
    "SA-BH",
    "SBH",
    "ABA",
    "SA-BA",
    "ABT",
    "SA-BT",
    "SNQ",
    "DNQ",
    "FE",
    "CE",
    "CEM",
    "SNB",
    "MNB",
    "WF",
    "HSSS",
    "PB",
    "CB",
    "FB",
    "SS",
    "HM",
    "FOA-CS",
    "FOA-SCS",
    "SN-DLS",
    "ZZSM",
    "Steam Iron",
    "Flatbed Iron",
    "Suction Iron",
    "Pressing Iron",
  ];

  useEffect(() => {
    const fetchGarmentTypes = async () => {
      try {
        const response = await fetch("/api/list-garment-types", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
        });
        if (!response.ok) {
          throw new Error("Failed to fetch garment types");
        }
        const data = await response.json();
        if (data.error) {
          throw new Error(data.error);
        }
        setGarmentTypes(data.garmentTypes || []);
      } catch (err) {
        console.error(err);
        setError("Failed to load garment types");
      }
    };
    fetchGarmentTypes();
  }, []);

  useEffect(() => {
    const fetchOperations = async () => {
      if (!formData.garmentType) return;
      try {
        const response = await fetch("/api/operation-bulletin", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            action: "listOperations",
            garmentTypeId: parseInt(formData.garmentType),
          }),
        });
        if (!response.ok) {
          throw new Error("Failed to fetch operations");
        }
        const data = await response.json();
        if (data.error) {
          throw new Error(data.error);
        }
        setOperations(
          (data.operations || []).map((op) => ({
            ...op,
            workers: "",
            smv: "",
            capacity40: "0.00",
            rate: "0.00",
            availableMachines: [],
          }))
        );
      } catch (err) {
        console.error(err);
        setError("Failed to load operations");
      }
    };
    fetchOperations();
  }, [formData.garmentType]);

  const handleInputChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value,
    });
  };

  const calculateCapacity40 = (smv) => {
    if (!smv) return "0.00";
    return ((60 / Number(smv)) * 0.4).toFixed(2);
  };

  const calculateCapacity = (workers, smv) => {
    if (!workers || !smv) return 0;
    return ((480 * Number(workers)) / Number(smv)).toFixed(2);
  };

  const calculateRate = (param1, param2) => {
    if (param2 === undefined) {
      if (!param1 || param1 === "0.00") return 0;
      return (90 / Number(param1)).toFixed(2);
    }
    if (!param1 || !param2) return 0;
    return ((480 * Number(param1)) / Number(param2) / 60).toFixed(2);
  };

  const handleOperationChange = async (index, field, value) => {
    const updatedOperations = [...operations];
    updatedOperations[index] = {
      ...updatedOperations[index],
      [field]: value,
    };

    if (field === "smv") {
      const capacity40 = calculateCapacity40(value);
      updatedOperations[index].capacity40 = capacity40;
      updatedOperations[index].rate = calculateRate(capacity40);
    }

    setOperations(updatedOperations);
  };

  const generatePDF = async () => {
    if (!formData.styleNumber || operations.length === 0) {
      setError("Please fill in style number and add at least one operation");
      return;
    }

    try {
      setLoading(true);
      setError(null);

      const totalRate = operations.reduce(
        (sum, op) => sum + (parseFloat(op.rate) || 0),
        0
      );

      const cmtBreakdown = {
        cutting: 10,
        stitching: totalRate * 1.15,
        finishing: 15,
        packing: 10,
      };
      cmtBreakdown.total =
        cmtBreakdown.cutting +
        cmtBreakdown.stitching +
        cmtBreakdown.finishing +
        cmtBreakdown.packing;

      const machineTypeSummary = operations.reduce((acc, op) => {
        if (op.machineType) {
          if (!acc[op.machineType]) {
            acc[op.machineType] = {
              count: 0,
              totalSMV: 0,
            };
          }
          acc[op.machineType].count += 1;
          acc[op.machineType].totalSMV += parseFloat(op.smv) || 0;
        }
        return acc;
      }, {});

      const groupedOperations = operations.reduce((acc, op) => {
        const section = op.section || "Other";
        if (!acc[section]) {
          acc[section] = [];
        }
        acc[section].push(op);
        return acc;
      }, {});

      const htmlContent = `
        <!DOCTYPE html>
        <html>
          <head>
            <meta charset="utf-8">
            <style>
              @page {
                size: A4;
                margin: 15mm;
              }
              body {
                font-family: Arial, sans-serif;
                line-height: 1.3;
                margin: 0;
                padding: 0;
                font-size: 11px;
              }
              h1, h2 {
                text-align: center;
                color: #333;
                margin: 8px 0;
              }
              h1 { font-size: 16px; }
              h2 { font-size: 14px; }
              .info {
                margin-bottom: 15px;
                border: 1px solid #000;
                padding: 8px;
              }
              table {
                width: 100%;
                border-collapse: collapse;
                margin-top: 8px;
                page-break-inside: auto;
              }
              tr {
                page-break-inside: avoid;
                page-break-after: auto;
              }
              th, td {
                border: 1px solid #000;
                padding: 4px;
                text-align: left;
                font-size: 10px;
              }
              th {
                background: #f0f0f0;
                font-weight: bold;
              }
              .section-header {
                background: #e0e0e0;
                font-weight: bold;
              }
              .center-align {
                text-align: center;
              }
              .right-align {
                text-align: right;
              }
              .summary-table {
                margin-top: 15px;
                width: 40%;
                float: left;
                margin-right: 5%;
              }
              .cmt-breakdown {
                margin-top: 15px;
                width: 40%;
                float: right;
              }
            </style>
          </head>
          <body>
            <h1>OPERATION BULLETIN</h1>
            
            <div class="info">
              <table>
                <tr>
                  <td><strong>Style Number:</strong> ${
                    formData.styleNumber
                  }</td>
                  <td><strong>Brand Name:</strong> ${
                    formData.brandName || "-"
                  }</td>
                </tr>
                <tr>
                  <td><strong>Order Quantity:</strong> ${
                    formData.orderQuantity || "-"
                  }</td>
                  <td><strong>Garment Type:</strong> ${
                    garmentTypes.find(
                      (g) => g.id === Number(formData.garmentType)
                    )?.name || "-"
                  }</td>
                </tr>
              </table>
            </div>

            ${Object.entries(groupedOperations)
              .map(
                ([section, sectionOps]) => `
              <div class="section-block">
                <h2>${section.toUpperCase()}</h2>
                <table>
                  <thead>
                    <tr>
                      <th style="width: 5%;">S.No</th>
                      <th style="width: 40%;">Operation Description</th>
                      <th style="width: 15%;">Machine</th>
                      <th style="width: 10%;">Workers</th>
                      <th style="width: 10%;">SMV</th>
                      <th style="width: 10%;">Capacity 40%</th>
                      <th style="width: 10%;">Rate</th>
                    </tr>
                  </thead>
                  <tbody>
                    ${sectionOps
                      .map(
                        (op, index) => `
                      <tr>
                        <td class="center-align">${index + 1}</td>
                        <td>${op.name || ""}</td>
                        <td class="center-align">${op.machineType || ""}</td>
                        <td class="center-align">${op.workers || ""}</td>
                        <td class="right-align">${op.smv || ""}</td>
                        <td class="right-align">${op.capacity40 || ""}</td>
                        <td class="right-align">${op.rate || ""}</td>
                      </tr>
                    `
                      )
                      .join("")}
                    <tr class="section-summary">
                      <td colspan="4" class="right-align"><strong>Section Total</strong></td>
                      <td class="right-align"><strong>${sectionOps
                        .reduce((sum, op) => sum + (parseFloat(op.smv) || 0), 0)
                        .toFixed(2)}</strong></td>
                      <td class="right-align"><strong>${sectionOps
                        .reduce(
                          (sum, op) => sum + (parseFloat(op.capacity40) || 0),
                          0
                        )
                        .toFixed(2)}</strong></td>
                      <td class="right-align"><strong>${sectionOps
                        .reduce(
                          (sum, op) => sum + (parseFloat(op.rate) || 0),
                          0
                        )
                        .toFixed(2)}</strong></td>
                    </tr>
                  </tbody>
                </table>
              </div>
            `
              )
              .join("")}

            <div style="clear: both;"></div>

            <div style="display: flex; justify-content: space-between; margin-top: 15px;">
              <div class="summary-table">
                <h2>MACHINE TYPE SUMMARY</h2>
                <table>
                  <thead>
                    <tr>
                      <th>Machine Type</th>
                      <th>Quantity</th>
                      <th>Total SMV</th>
                    </tr>
                  </thead>
                  <tbody>
                    ${Object.entries(machineTypeSummary)
                      .map(
                        ([machine, data]) => `
                      <tr>
                        <td>${machine}</td>
                        <td class="center-align">${data.count}</td>
                        <td class="right-align">${data.totalSMV.toFixed(2)}</td>
                      </tr>
                    `
                      )
                      .join("")}
                    <tr>
                      <td colspan="2" class="right-align"><strong>Total SMV</strong></td>
                      <td class="right-align"><strong>${Object.values(
                        machineTypeSummary
                      )
                        .reduce((sum, data) => sum + data.totalSMV, 0)
                        .toFixed(2)}</strong></td>
                    </tr>
                  </tbody>
                </table>
              </div>

              <div class="cmt-breakdown">
                <h2>CMT BREAKDOWN</h2>
                <table>
                  <thead>
                    <tr>
                      <th>Operation</th>
                      <th>Rate (Rs.)</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr>
                      <td>Cutting</td>
                      <td class="right-align">${cmtBreakdown.cutting.toFixed(
                        2
                      )}</td>
                    </tr>
                    <tr>
                      <td>Stitching</td>
                      <td class="right-align">${cmtBreakdown.stitching.toFixed(
                        2
                      )}</td>
                    </tr>
                    <tr>
                      <td>Finishing</td>
                      <td class="right-align">${cmtBreakdown.finishing.toFixed(
                        2
                      )}</td>
                    </tr>
                    <tr>
                      <td>Packing</td>
                      <td class="right-align">${cmtBreakdown.packing.toFixed(
                        2
                      )}</td>
                    </tr>
                    <tr>
                      <td><strong>Total CMT</strong></td>
                      <td class="right-align"><strong>${cmtBreakdown.total.toFixed(
                        2
                      )}</strong></td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>

            <div style="margin-top: 15px;">
              <p style="font-size: 10px;">Generated: ${new Date().toLocaleDateString()}</p>
            </div>
          </body>
        </html>
      `.trim();

      const response = await fetch("/api/html2pdf", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          html: htmlContent,
        }),
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(
          errorData.error || `Failed to generate PDF: ${response.statusText}`
        );
      }

      const data = await response.json();

      if (!data.success || !data.pdf) {
        throw new Error(data.error || "Failed to generate PDF");
      }

      const saveResponse = await fetch("/api/operation-bulletin", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          action: "create",
          styleNumber: formData.styleNumber,
          brandName: formData.brandName,
          orderQuantity: parseInt(formData.orderQuantity) || null,
          garmentTypeId: parseInt(formData.garmentType),
          operations: operations.map((op) => ({
            name: op.name,
            category: op.category,
            machineType: op.machineType,
            workers: parseInt(op.workers) || null,
            smv: parseFloat(op.smv) || null,
            capacity40: parseFloat(op.capacity40) || null,
            rate: parseFloat(op.rate) || null,
          })),
        }),
      });

      if (!saveResponse.ok) {
        throw new Error("Failed to save Operation Bulletin");
      }

      const byteCharacters = atob(data.pdf);
      const byteNumbers = new Array(byteCharacters.length);
      for (let i = 0; i < byteCharacters.length; i++) {
        byteNumbers[i] = byteCharacters.charCodeAt(i);
      }
      const byteArray = new Uint8Array(byteNumbers);
      const blob = new Blob([byteArray], { type: "application/pdf" });

      const url = window.URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.style.display = "none";
      a.href = url;
      a.download = `operation_bulletin_${formData.styleNumber}.pdf`;

      document.body.appendChild(a);
      a.click();

      window.URL.revokeObjectURL(url);
      document.body.removeChild(a);

      setFormData({
        styleNumber: "",
        brandName: "",
        orderQuantity: "",
        garmentType: "",
      });
      setOperations([]);
    } catch (err) {
      console.error("Error:", err);
      setError(
        err.message || "Could not complete the operation. Please try again."
      );
    } finally {
      setLoading(false);
    }
  };

  const removeOperation = (indexToRemove) => {
    setOperations(operations.filter((_, index) => index !== indexToRemove));
  };

  const addOperation = () => {
    if (!selectedSection) {
      setError("Please enter a section name");
      return;
    }

    setOperations([
      ...operations,
      {
        name: "",
        section: selectedSection,
        machineType: "",
        workers: "",
        smv: "",
        capacity40: "0.00",
        rate: "0.00",
        availableMachines: [],
      },
    ]);
  };

  const groupedOperations = operations.reduce((acc, op) => {
    const section = op.section || "unsorted";
    if (!acc[section]) {
      acc[section] = [];
    }
    acc[section].push(op);
    return acc;
  }, {});

  const handleAddGarmentType = async () => {
    if (!newGarmentType.trim()) {
      setError("Please enter a garment type name");
      return;
    }

    try {
      setAddingGarmentType(true);
      const response = await fetch("/api/add-garment-type", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name: newGarmentType.trim() }),
      });

      if (!response.ok) {
        throw new Error("Failed to add garment type");
      }

      const data = await response.json();
      if (data.error) {
        throw new Error(data.error);
      }

      const updatedTypes = [...garmentTypes, data.garmentType];
      setGarmentTypes(updatedTypes);
      setNewGarmentType("");
      setShowAddGarmentType(false);
      setError(null);
    } catch (err) {
      console.error(err);
      setError("Failed to add garment type");
    } finally {
      setAddingGarmentType(false);
    }
  };

  const handleAddMachineType = async () => {
    if (!newMachineType.name.trim() || !newMachineType.category.trim()) {
      setError("Please enter both machine name and category");
      return;
    }

    try {
      setAddingMachineType(true);
      const response = await fetch("/api/add-machine-type", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          name: newMachineType.name.trim(),
          category: newMachineType.category.trim(),
        }),
      });

      if (!response.ok) {
        throw new Error("Failed to add machine type");
      }

      const data = await response.json();
      if (data.error) {
        throw new Error(data.error);
      }

      const updatedMachineTypes = [...machineTypes, data.machineType.name];
      machineTypes.push(data.machineType.name);
      setNewMachineType({ name: "", category: "" });
      setShowAddMachineType(false);
      setError(null);
    } catch (err) {
      console.error(err);
      setError("Failed to add machine type");
    } finally {
      setAddingMachineType(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 p-4 md:p-8">
      <div className="max-w-7xl mx-auto bg-white rounded-xl shadow-lg p-6">
        <div className="flex justify-between items-center mb-8">
          <h1 className="text-3xl font-bold text-gray-900">
            Operation Bulletin Generator
          </h1>
          <a
            href="/history"
            className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 transition-colors"
          >
            View History
          </a>
        </div>

        <div className="bg-gray-50 rounded-lg p-6 mb-8">
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
            <div className="space-y-2">
              <label className="text-sm font-semibold text-gray-700 block">
                Style Number *
              </label>
              <input
                type="text"
                name="styleNumber"
                value={formData.styleNumber}
                onChange={handleInputChange}
                className="w-full rounded-lg border-2 border-gray-300 p-3 focus:border-blue-500 focus:ring focus:ring-blue-200 transition duration-200"
                placeholder="Enter style number"
              />
            </div>

            <div className="space-y-2">
              <label className="text-sm font-semibold text-gray-700 block">
                Brand Name
              </label>
              <input
                type="text"
                name="brandName"
                value={formData.brandName}
                onChange={handleInputChange}
                className="w-full rounded-lg border-2 border-gray-300 p-3 focus:border-blue-500 focus:ring focus:ring-blue-200 transition duration-200"
                placeholder="Enter brand name"
              />
            </div>

            <div className="space-y-2">
              <label className="text-sm font-semibold text-gray-700 block">
                Order Quantity
              </label>
              <input
                type="number"
                name="orderQuantity"
                value={formData.orderQuantity}
                onChange={handleInputChange}
                className="w-full rounded-lg border-2 border-gray-300 p-3 focus:border-blue-500 focus:ring focus:ring-blue-200 transition duration-200"
                placeholder="Enter quantity"
              />
            </div>

            <div className="space-y-2">
              <label className="text-sm font-semibold text-gray-700 block">
                Garment Type *
              </label>
              <div className="flex gap-2">
                <select
                  name="garmentType"
                  value={formData.garmentType}
                  onChange={handleInputChange}
                  className="w-full rounded-lg border-2 border-gray-300 p-3 focus:border-blue-500 focus:ring focus:ring-blue-200 transition duration-200"
                >
                  <option value="">Select Type</option>
                  {garmentTypes.map((type) => (
                    <option key={type.id} value={type.id}>
                      {type.name}
                    </option>
                  ))}
                </select>
                <button
                  onClick={() => setShowAddGarmentType(true)}
                  className="px-3 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors"
                  title="Add new garment type"
                >
                  ➕
                </button>
              </div>
            </div>
          </div>
        </div>

        <div className="bg-white rounded-lg shadow mb-6">
          <div className="overflow-x-auto">
            <table className="w-full border-collapse">
              <thead>
                <tr className="bg-gray-50">
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Operation
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Machine
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Workers
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    SMV
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Capacity 40%
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Rate
                  </th>
                  <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                    Actions
                  </th>
                </tr>
              </thead>
              <tbody>
                {operations.map((operation, index) => (
                  <tr
                    key={index}
                    className="hover:bg-gray-50 transition duration-150"
                  >
                    <td className="border border-gray-300 p-4">
                      <input
                        type="text"
                        value={operation.name}
                        onChange={(e) =>
                          handleOperationChange(index, "name", e.target.value)
                        }
                        className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                        placeholder="Operation name"
                      />
                    </td>
                    <td className="border border-gray-300 p-4">
                      <div className="flex gap-2">
                        <select
                          value={operation.machineType}
                          onChange={(e) =>
                            handleOperationChange(
                              index,
                              "machineType",
                              e.target.value
                            )
                          }
                          className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                        >
                          <option value="">Select Machine</option>
                          {machineTypes.map((machine) => (
                            <option key={machine} value={machine}>
                              {machine}
                            </option>
                          ))}
                        </select>
                        <button
                          onClick={() => setShowAddMachineType(true)}
                          className="px-2 py-1 bg-green-600 text-white rounded hover:bg-green-700 transition-colors"
                          title="Add new machine type"
                        >
                          ➕
                        </button>
                      </div>
                    </td>
                    <td className="border border-gray-300 p-4">
                      <input
                        type="number"
                        value={operation.workers}
                        onChange={(e) =>
                          handleOperationChange(
                            index,
                            "workers",
                            e.target.value
                          )
                        }
                        className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                        placeholder="Workers"
                      />
                    </td>
                    <td className="border border-gray-300 p-4">
                      <input
                        type="number"
                        step="0.01"
                        value={operation.smv}
                        onChange={(e) =>
                          handleOperationChange(index, "smv", e.target.value)
                        }
                        className="w-full bg-transparent rounded p-2 focus:outline-none focus:ring-2 focus:ring-blue-200"
                        placeholder="Enter SMV"
                        style={{ minWidth: "120px" }}
                      />
                    </td>
                    <td className="border border-gray-300 p-4 text-center font-medium text-lg">
                      {operation.capacity40}
                    </td>
                    <td className="border border-gray-300 p-4 text-center">
                      {operation.rate}
                    </td>
                    <td className="border border-gray-300 p-4">
                      <button
                        onClick={() => removeOperation(index)}
                        className="flex items-center justify-center w-full px-3 py-1.5 text-red-600 hover:bg-red-50 rounded-md transition-colors"
                        title="Remove operation"
                      >
                        🗑️ Remove
                      </button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>

        <div className="mb-6 flex gap-4 items-end">
          <div className="flex-1 max-w-xs">
            <label className="text-sm font-semibold text-gray-700 block mb-2">
              Section Name
            </label>
            <input
              type="text"
              value={selectedSection}
              onChange={(e) => setSelectedSection(e.target.value)}
              className="w-full rounded-lg border-2 border-gray-300 p-3 focus:border-blue-500 focus:ring focus:ring-blue-200 transition duration-200"
              placeholder="Enter section name"
            />
          </div>
          <button
            onClick={addOperation}
            className="bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700 transition-colors flex items-center gap-2"
          >
            <span>➕</span> Add Operation
          </button>
        </div>

        {sections.map((section) => {
          const sectionOperations = operations.filter(
            (op) => op.section === section.id
          );
          if (sectionOperations.length === 0) return null;

          return (
            <div key={section.id} className="mb-8">
              <div className="flex items-center justify-between mb-4">
                <h2 className="text-xl font-semibold text-gray-800">
                  {section.name} Section ({sectionOperations.length}/
                  {section.maxOps} Operations)
                </h2>
              </div>
              <div className="bg-white rounded-lg shadow overflow-x-auto">
                <table className="w-full border-collapse">
                  <thead>
                    <tr className="bg-gray-50">
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Sr.
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Operation
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Machine
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Workers
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        SMV
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Capacity 40%
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Rate
                      </th>
                      <th className="border border-gray-300 p-4 text-left text-sm font-semibold text-gray-700">
                        Actions
                      </th>
                    </tr>
                  </thead>
                  <tbody>
                    {sectionOperations.map((operation, index) => (
                      <tr
                        key={index}
                        className="hover:bg-gray-50 transition duration-150"
                      >
                        <td className="border border-gray-300 p-4 text-center">
                          {index + 1}
                        </td>
                        <td className="border border-gray-300 p-4">
                          <input
                            type="text"
                            value={operation.name}
                            onChange={(e) =>
                              handleOperationChange(
                                operations.indexOf(operation),
                                "name",
                                e.target.value
                              )
                            }
                            className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                            placeholder="Operation name"
                          />
                        </td>
                        <td className="border border-gray-300 p-4">
                          <select
                            value={operation.machineType}
                            onChange={(e) =>
                              handleOperationChange(
                                operations.indexOf(operation),
                                "machineType",
                                e.target.value
                              )
                            }
                            className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                          >
                            <option value="">Select Machine</option>
                            {operation.availableMachines &&
                              operation.availableMachines.map((machine) => (
                                <option key={machine.id} value={machine.name}>
                                  {machine.name}
                                </option>
                              ))}
                          </select>
                        </td>
                        <td className="border border-gray-300 p-4">
                          <input
                            type="number"
                            value={operation.workers}
                            onChange={(e) =>
                              handleOperationChange(
                                operations.indexOf(operation),
                                "workers",
                                e.target.value
                              )
                            }
                            className="w-full bg-transparent rounded p-1 focus:outline-none focus:ring-2 focus:ring-blue-200"
                            placeholder="Workers"
                          />
                        </td>
                        <td className="border border-gray-300 p-4">
                          <input
                            type="number"
                            step="0.01"
                            value={operation.smv}
                            onChange={(e) =>
                              handleOperationChange(
                                operations.indexOf(operation),
                                "smv",
                                e.target.value
                              )
                            }
                            className="w-full bg-transparent rounded p-2 focus:outline-none focus:ring-2 focus:ring-blue-200"
                            placeholder="Enter SMV"
                          />
                        </td>
                        <td className="border border-gray-300 p-4 text-center font-medium">
                          {operation.capacity40}
                        </td>
                        <td className="border border-gray-300 p-4 text-center">
                          {operation.rate}
                        </td>
                        <td className="border border-gray-300 p-4">
                          <button
                            onClick={() =>
                              removeOperation(operations.indexOf(operation))
                            }
                            className="flex items-center justify-center w-full px-3 py-1.5 text-red-600 hover:bg-red-50 rounded-md transition-colors"
                            title="Remove operation"
                          >
                            🗑️ Remove
                          </button>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          );
        })}

        <div className="mt-6 flex justify-end">
          <button
            onClick={generatePDF}
            disabled={
              loading || !formData.styleNumber || operations.length === 0
            }
            className="bg-green-600 text-white px-6 py-3 rounded-lg hover:bg-green-700 transition-colors disabled:bg-gray-400 flex items-center gap-2"
          >
            {loading ? (
              <>
                <span className="animate-spin">⏳</span>
                Generating...
              </>
            ) : (
              <>
                <span>📄</span> Generate PDF
              </>
            )}
          </button>
        </div>

        {error && (
          <div className="mt-4 p-4 bg-red-50 text-red-600 rounded-lg border border-red-200">
            ⚠️ {error}
          </div>
        )}

        {showAddGarmentType && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4">
            <div className="bg-white rounded-xl p-6 max-w-md w-full">
              <h3 className="text-xl font-semibold mb-4">
                Add New Garment Type
              </h3>
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Garment Type Name
                  </label>
                  <input
                    type="text"
                    value={newGarmentType}
                    onChange={(e) => setNewGarmentType(e.target.value)}
                    className="w-full rounded-lg border-2 border-gray-300 p-2 focus:border-blue-500 focus:ring focus:ring-blue-200"
                    placeholder="Enter garment type name"
                  />
                </div>
                <div className="flex justify-end gap-2">
                  <button
                    onClick={() => setShowAddGarmentType(false)}
                    className="px-4 py-2 text-gray-600 hover:text-gray-800"
                  >
                    Cancel
                  </button>
                  <button
                    onClick={handleAddGarmentType}
                    disabled={addingGarmentType}
                    className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors disabled:bg-gray-400"
                  >
                    {addingGarmentType ? "Adding..." : "Add Garment Type"}
                  </button>
                </div>
              </div>
            </div>
          </div>
        )}

        {showAddMachineType && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4">
            <div className="bg-white rounded-xl p-6 max-w-md w-full">
              <h3 className="text-xl font-semibold mb-4">
                Add New Machine Type
              </h3>
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Machine Name
                  </label>
                  <input
                    type="text"
                    value={newMachineType.name}
                    onChange={(e) =>
                      setNewMachineType({
                        ...newMachineType,
                        name: e.target.value,
                      })
                    }
                    className="w-full rounded-lg border-2 border-gray-300 p-2 focus:border-blue-500 focus:ring focus:ring-blue-200"
                    placeholder="Enter machine name"
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Category
                  </label>
                  <input
                    type="text"
                    value={newMachineType.category}
                    onChange={(e) =>
                      setNewMachineType({
                        ...newMachineType,
                        category: e.target.value,
                      })
                    }
                    className="w-full rounded-lg border-2 border-gray-300 p-2 focus:border-blue-500 focus:ring focus:ring-blue-200"
                    placeholder="Enter machine category"
                  />
                </div>
                <div className="flex justify-end gap-2">
                  <button
                    onClick={() => setShowAddMachineType(false)}
                    className="px-4 py-2 text-gray-600 hover:text-gray-800"
                  >
                    Cancel
                  </button>
                  <button
                    onClick={handleAddMachineType}
                    disabled={addingMachineType}
                    className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors disabled:bg-gray-400"
                  >
                    {addingMachineType ? "Adding..." : "Add Machine Type"}
                  </button>
                </div>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

export default MainComponent;
