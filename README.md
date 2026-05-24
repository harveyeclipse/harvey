using System;
using System.IO;
using System.Linq;
using System.Collections.Generic;
using System.Security.Cryptography;
using System.Text;

namespace VehicleServiceRecords
{
    class Program
    {
        static string dataFolder = "Data";
        static string recordsFile = Path.Combine(dataFolder, "records.txt");
        static string auditFile = Path.Combine(dataFolder, "audit.txt");
        static string reportsFolder = Path.Combine(dataFolder, "reports");

        static void Main(string[] args)
        {
            InitializeStorage();

            while (true)
            {
                Console.Clear();
                Console.WriteLine("==== VEHICLE SERVICE RECORD SYSTEM ====");
                Console.WriteLine("1. Add Record");
                Console.WriteLine("2. View Records");
                Console.WriteLine("3. Search Record");
                Console.WriteLine("4. Update Record");
                Console.WriteLine("5. Soft Delete Record");
                Console.WriteLine("6. Hard Delete Record");
                Console.WriteLine("7. Generate Report");
                Console.WriteLine("8. Exit");
                Console.Write("Choose option: ");

                string choice = Console.ReadLine();

                switch (choice)
                {
                    case "1":
                        AddRecord();
                        break;
                    case "2":
                        ViewRecords();
                        break;
                    case "3":
                        SearchRecord();
                        break;
                    case "4":
                        UpdateRecord();
                        break;
                    case "5":
                        SoftDeleteRecord();
                        break;
                    case "6":
                        HardDeleteRecord();
                        break;
                    case "7":
                        GenerateReport();
                        break;
                    case "8":
                        return;
                    default:
                        Console.WriteLine("Invalid choice.");
                        Pause();
                        break;
                }
            }
        }

        // ================= INITIALIZE =================

        static void InitializeStorage()
        {
            try
            {
                if (!Directory.Exists(dataFolder))
                    Directory.CreateDirectory(dataFolder);

                if (!Directory.Exists(reportsFolder))
                    Directory.CreateDirectory(reportsFolder);

                if (!File.Exists(recordsFile))
                    File.Create(recordsFile).Close();

                if (!File.Exists(auditFile))
                    File.Create(auditFile).Close();

                LogAction("SYSTEM", "Storage initialized.");
            }
            catch (Exception ex)
            {
                Console.WriteLine("Initialization Error: " + ex.Message);
            }
        }

        // ================= MODEL =================

        class VehicleRecord
        {
            public string RecordId { get; set; }
            public string VehicleOwner { get; set; }
            public string VehicleModel { get; set; }
            public string PlateNumber { get; set; }
            public string ServiceType { get; set; }
            public double ServiceCost { get; set; }
            public DateTime CreatedAt { get; set; }
            public DateTime UpdatedAt { get; set; }
            public bool IsActive { get; set; }
            public string Checksum { get; set; }

            public override string ToString()
            {
                return $"{RecordId}|{VehicleOwner}|{VehicleModel}|{PlateNumber}|{ServiceType}|{ServiceCost}|{CreatedAt}|{UpdatedAt}|{IsActive}|{Checksum}";
            }

            public static VehicleRecord FromString(string line)
            {
                string[] parts = line.Split('|');

                if (parts.Length != 10)
                    throw new Exception("Malformed record.");

                return new VehicleRecord
                {
                    RecordId = parts[0],
                    VehicleOwner = parts[1],
                    VehicleModel = parts[2],
                    PlateNumber = parts[3],
                    ServiceType = parts[4],
                    ServiceCost = double.Parse(parts[5]),
                    CreatedAt = DateTime.Parse(parts[6]),
                    UpdatedAt = DateTime.Parse(parts[7]),
                    IsActive = bool.Parse(parts[8]),
                    Checksum = parts[9]
                };
            }
        }

        // ================= ADD =================

        static void AddRecord()
        {
            try
            {
                VehicleRecord record = new VehicleRecord();

                record.RecordId = Guid.NewGuid().ToString();

                Console.Write("Vehicle Owner: ");
                record.VehicleOwner = ValidateRequired(Console.ReadLine());

                Console.Write("Vehicle Model: ");
                record.VehicleModel = ValidateRequired(Console.ReadLine());

                Console.Write("Plate Number: ");
                record.PlateNumber = ValidateRequired(Console.ReadLine());

                Console.Write("Service Type: ");
                record.ServiceType = ValidateRequired(Console.ReadLine());

                Console.Write("Service Cost: ");
                record.ServiceCost = ValidateDouble(Console.ReadLine());

                record.CreatedAt = DateTime.Now;
                record.UpdatedAt = DateTime.Now;
                record.IsActive = true;

                record.Checksum = GenerateChecksum(record);

                File.AppendAllText(recordsFile, record + Environment.NewLine);

                LogAction("ADD", $"Added record {record.RecordId}");

                Console.WriteLine("Record added successfully.");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine("Error: " + ex.Message);
            }

            Pause();
        }

        // ================= VIEW =================

        static void ViewRecords()
        {
            try
            {
                List<VehicleRecord> records = LoadRecords();

                var activeRecords = records.Where(r => r.IsActive);

                Console.WriteLine("\n==== ACTIVE RECORDS ====");

                foreach (var r in activeRecords)
                {
                    DisplayRecord(r);
                }

                LogAction("READ", "Viewed records.");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= SEARCH =================

        static void SearchRecord()
        {
            try
            {
                Console.Write("Enter Plate Number: ");
                string plate = Console.ReadLine();

                var records = LoadRecords()
                    .Where(r => r.PlateNumber.Contains(plate, StringComparison.OrdinalIgnoreCase)
                             && r.IsActive);

                foreach (var r in records)
                {
                    DisplayRecord(r);
                }

                LogAction("READ", $"Searched plate {plate}");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= UPDATE =================

        static void UpdateRecord()
        {
            try
            {
                List<VehicleRecord> records = LoadRecords();

                Console.Write("Enter Record ID: ");
                string id = Console.ReadLine();

                VehicleRecord record = records.FirstOrDefault(r => r.RecordId == id);

                if (record == null)
                {
                    Console.WriteLine("Record not found.");
                    Pause();
                    return;
                }

                Console.Write("New Service Type: ");
                record.ServiceType = ValidateRequired(Console.ReadLine());

                Console.Write("New Service Cost: ");
                record.ServiceCost = ValidateDouble(Console.ReadLine());

                record.UpdatedAt = DateTime.Now;
                record.Checksum = GenerateChecksum(record);

                SaveAllRecords(records);

                LogAction("UPDATE", $"Updated record {id}");

                Console.WriteLine("Record updated.");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= SOFT DELETE =================

        static void SoftDeleteRecord()
        {
            try
            {
                List<VehicleRecord> records = LoadRecords();

                Console.Write("Enter Record ID: ");
                string id = Console.ReadLine();

                VehicleRecord record = records.FirstOrDefault(r => r.RecordId == id);

                if (record == null)
                {
                    Console.WriteLine("Record not found.");
                    Pause();
                    return;
                }

                record.IsActive = false;
                record.UpdatedAt = DateTime.Now;

                SaveAllRecords(records);

                LogAction("DELETE", $"Soft deleted {id}");

                Console.WriteLine("Record soft deleted.");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= HARD DELETE =================

        static void HardDeleteRecord()
        {
            try
            {
                List<VehicleRecord> records = LoadRecords();

                Console.Write("Enter Record ID: ");
                string id = Console.ReadLine();

                records = records.Where(r => r.RecordId != id).ToList();

                SaveAllRecords(records);

                LogAction("DELETE", $"Hard deleted {id}");

                Console.WriteLine("Record permanently deleted.");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= REPORT =================

        static void GenerateReport()
        {
            try
            {
                var records = LoadRecords().Where(r => r.IsActive);

                double total = records.Sum(r => r.ServiceCost);

                string report =
                    "==== VEHICLE SERVICE REPORT ====\n" +
                    $"Generated: {DateTime.Now}\n" +
                    $"Total Active Records: {records.Count()}\n" +
                    $"Total Service Revenue: {total}\n";

                string fileName = $"Report_{DateTime.Now:yyyyMMddHHmmss}.txt";

                string path = Path.Combine(reportsFolder, fileName);

                File.WriteAllText(path, report);

                LogAction("REPORT", $"Generated {fileName}");

                Console.WriteLine("Report generated.");
                Console.WriteLine($"Saved to: {path}");
            }
            catch (Exception ex)
            {
                LogAction("ERROR", ex.Message);
                Console.WriteLine(ex.Message);
            }

            Pause();
        }

        // ================= FILE HANDLING =================

        static List<VehicleRecord> LoadRecords()
        {
            List<VehicleRecord> records = new List<VehicleRecord>();

            string[] lines = File.ReadAllLines(recordsFile);

            foreach (string line in lines)
            {
                try
                {
                    if (!string.IsNullOrWhiteSpace(line))
                    {
                        records.Add(VehicleRecord.FromString(line));
                    }
                }
                catch
                {
                    LogAction("ERROR", "Malformed record skipped.");
                }
            }

            return records;
        }

        static void SaveAllRecords(List<VehicleRecord> records)
        {
            File.WriteAllLines(recordsFile, records.Select(r => r.ToString()));
        }

        // ================= VALIDATION =================

        static string ValidateRequired(string input)
        {
            if (string.IsNullOrWhiteSpace(input))
                throw new Exception("Field cannot be empty.");

            return input;
        }

        static double ValidateDouble(string input)
        {
            if (!double.TryParse(input, out double result))
                throw new Exception("Invalid numeric value.");

            return result;
        }

        // ================= CHECKSUM =================

        static string GenerateChecksum(VehicleRecord r)
        {
            string raw =
                r.RecordId +
                r.VehicleOwner +
                r.VehicleModel +
                r.PlateNumber +
                r.ServiceType +
                r.ServiceCost;

            using (SHA256 sha = SHA256.Create())
            {
                byte[] bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(raw));

                return Convert.ToBase64String(bytes);
            }
        }

        // ================= AUDIT =================

        static void LogAction(string action, string details)
        {
            string log =
                $"{DateTime.Now} | {action} | {details}";

            File.AppendAllText(auditFile, log + Environment.NewLine);
        }

        // ================= DISPLAY =================

        static void DisplayRecord(VehicleRecord r)
        {
            Console.WriteLine("----------------------------------");
            Console.WriteLine($"ID: {r.RecordId}");
            Console.WriteLine($"Owner: {r.VehicleOwner}");
            Console.WriteLine($"Model: {r.VehicleModel}");
            Console.WriteLine($"Plate: {r.PlateNumber}");
            Console.WriteLine($"Service: {r.ServiceType}");
            Console.WriteLine($"Cost: {r.ServiceCost}");
            Console.WriteLine($"Created: {r.CreatedAt}");
            Console.WriteLine($"Updated: {r.UpdatedAt}");
            Console.WriteLine($"Active: {r.IsActive}");
        }

        static void Pause()
        {
            Console.WriteLine("\nPress any key to continue...");
            Console.ReadKey();
        }
    }
}
