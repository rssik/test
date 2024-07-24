# test
当然，我很乐意为您提供这些方法的具体实现。让我们逐一展示这些关键方法的代码。



```csharp
using System;
using System.IO;
using System.Drawing;
using System.Drawing.Imaging;
using System.Collections.Generic;
using System.Linq;
using System.Data.SQLite;
using System.Globalization;

public partial class MainForm
{
    private static string connectionString = "Data Source=KlarfDatabase.sqlite;Version=3;";

    private void InitializeDatabase()
    {
        using (var connection = new SQLiteConnection(connectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                // Create KlarfData table
                command.CommandText = @"
                    CREATE TABLE IF NOT EXISTS KlarfData (
                        WaferID TEXT PRIMARY KEY,
                        FileTimestamp TEXT,
                        InspectionStationID TEXT,
                        SampleSize INTEGER,
                        LotID TEXT,
                        DeviceID TEXT,
                        SetupID TEXT,
                        StepID TEXT,
                        SampleOrientationMarkType TEXT
                    )";
                command.ExecuteNonQuery();

                // Create Defects table
                command.CommandText = @"
                    CREATE TABLE IF NOT EXISTS Defects (
                        ID INTEGER PRIMARY KEY AUTOINCREMENT,
                        WaferID TEXT,
                        X REAL,
                        Y REAL,
                        Size REAL,
                        ClassNumber INTEGER,
                        FOREIGN KEY(WaferID) REFERENCES KlarfData(WaferID)
                    )";
                command.ExecuteNonQuery();
            }
        }
    }

    private KlarfData ParseKlarfFile(string filePath)
    {
        KlarfData klarfData = new KlarfData();
        List<string> defectRecordSpec = new List<string>();

        using (StreamReader reader = new StreamReader(filePath))
        {
            string line;
            string currentSection = "";

            while ((line = reader.ReadLine()) != null)
            {
                line = line.Trim();

                if (line.StartsWith("FileTimestamp"))
                    klarfData.FileTimestamp = ParseValue(line);
                else if (line.StartsWith("InspectionStationID"))
                    klarfData.InspectionStationID = ParseValue(line);
                else if (line.StartsWith("SampleSize"))
                    klarfData.SampleSize = int.Parse(ParseValue(line));
                else if (line.StartsWith("WaferID"))
                    klarfData.WaferID = ParseValue(line);
                else if (line.StartsWith("LotID"))
                    klarfData.LotID = ParseValue(line);
                else if (line.StartsWith("DeviceID"))
                    klarfData.DeviceID = ParseValue(line);
                else if (line.StartsWith("SetupID"))
                    klarfData.SetupID = ParseValue(line);
                else if (line.StartsWith("StepID"))
                    klarfData.StepID = ParseValue(line);
                else if (line.StartsWith("SampleOrientationMarkType"))
                    klarfData.SampleOrientationMarkType = ParseValue(line);
                else if (line.StartsWith("DefectRecordSpec"))
                    defectRecordSpec = ParseDefectRecordSpec(line);
                else if (line.StartsWith("DefectList"))
                    currentSection = "DefectList";
                else if (currentSection == "DefectList" && !string.IsNullOrWhiteSpace(line))
                    klarfData.Defects.Add(ParseDefectData(line, defectRecordSpec));
                else if (string.IsNullOrWhiteSpace(line))
                    currentSection = "";
            }
        }

        return klarfData;
    }

    private void SaveToDatabase(KlarfData klarfData)
    {
        using (var connection = new SQLiteConnection(connectionString))
        {
            connection.Open();
            using (var transaction = connection.BeginTransaction())
            {
                try
                {
                    using (var command = new SQLiteCommand(connection))
                    {
                        // Insert KlarfData
                        command.CommandText = @"
                            INSERT OR REPLACE INTO KlarfData 
                            (WaferID, FileTimestamp, InspectionStationID, SampleSize, LotID, DeviceID, SetupID, StepID, SampleOrientationMarkType)
                            VALUES (@WaferID, @FileTimestamp, @InspectionStationID, @SampleSize, @LotID, @DeviceID, @SetupID, @StepID, @SampleOrientationMarkType)";
                        command.Parameters.AddWithValue("@WaferID", klarfData.WaferID);
                        command.Parameters.AddWithValue("@FileTimestamp", klarfData.FileTimestamp);
                        command.Parameters.AddWithValue("@InspectionStationID", klarfData.InspectionStationID);
                        command.Parameters.AddWithValue("@SampleSize", klarfData.SampleSize);
                        command.Parameters.AddWithValue("@LotID", klarfData.LotID);
                        command.Parameters.AddWithValue("@DeviceID", klarfData.DeviceID);
                        command.Parameters.AddWithValue("@SetupID", klarfData.SetupID);
                        command.Parameters.AddWithValue("@StepID", klarfData.StepID);
                        command.Parameters.AddWithValue("@SampleOrientationMarkType", klarfData.SampleOrientationMarkType);
                        command.ExecuteNonQuery();

                        // Insert Defects
                        command.CommandText = @"
                            INSERT INTO Defects (WaferID, X, Y, Size, ClassNumber)
                            VALUES (@WaferID, @X, @Y, @Size, @ClassNumber)";
                        foreach (var defect in klarfData.Defects)
                        {
                            command.Parameters.Clear();
                            command.Parameters.AddWithValue("@WaferID", klarfData.WaferID);
                            command.Parameters.AddWithValue("@X", defect.X);
                            command.Parameters.AddWithValue("@Y", defect.Y);
                            command.Parameters.AddWithValue("@Size", defect.Size);
                            command.Parameters.AddWithValue("@ClassNumber", defect.ClassNumber);
                            command.ExecuteNonQuery();
                        }
                    }
                    transaction.Commit();
                }
                catch
                {
                    transaction.Rollback();
                    throw;
                }
            }
        }
    }

    private void GenerateWaferMap(string waferID, string outputPath)
    {
        KlarfData klarfData = RetrieveKlarfDataFromDatabase(waferID);
        List<DefectData> defects = RetrieveDefectsFromDatabase(waferID);

        int imageSize = 1000;
        float maxCoordinate = defects.Count > 0 ? defects.Max(d => Math.Max(Math.Abs(d.X), Math.Abs(d.Y))) : 1;
        float scale = imageSize / (2 * maxCoordinate);

        using (Bitmap bitmap = new Bitmap(imageSize, imageSize))
        using (Graphics g = Graphics.FromImage(bitmap))
        {
            g.Clear(Color.LightGray);
            g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;

            // Draw wafer outline
            g.DrawEllipse(Pens.Black, 0, 0, imageSize - 1, imageSize - 1);

            // Draw orientation mark
            DrawOrientationMark(g, klarfData.SampleOrientationMarkType, imageSize);

            // Draw defects
            foreach (var defect in defects)
            {
                float x = (defect.X * scale) + (imageSize / 2);
                float y = (defect.Y * scale) + (imageSize / 2);
                float size = Math.Max(2, defect.Size * scale);

                Color defectColor = GetDefectColor(defect.ClassNumber);
                g.FillEllipse(new SolidBrush(defectColor), x - (size / 2), y - (size / 2), size, size);
            }

            // Add metadata
            DrawMetadata(g, klarfData, defects.Count);

            bitmap.Save(outputPath, ImageFormat.Png);
        }
    }

    private string ParseValue(string line)
    {
        return line.Split(new[] { ' ' }, 2, StringSplitOptions.RemoveEmptyEntries)[1].Trim();
    }

    private List<string> ParseDefectRecordSpec(string line)
    {
        string spec = line.Split(new[] { ' ' }, 2, StringSplitOptions.RemoveEmptyEntries)[1];
        return spec.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries).ToList();
    }

    private DefectData ParseDefectData(string line, List<string> defectRecordSpec)
    {
        string[] values = line.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
        DefectData defect = new DefectData();

        for (int i = 0; i < Math.Min(values.Length, defectRecordSpec.Count); i++)
        {
            switch (defectRecordSpec[i].ToLower())
            {
                case "xrel":
                    defect.X = float.Parse(values[i], CultureInfo.InvariantCulture);
                    break;
                case "yrel":
                    defect.Y = float.Parse(values[i], CultureInfo.InvariantCulture);
                    break;
                case "dsize":
                    defect.Size = float.Parse(values[i], CultureInfo.InvariantCulture);
                    break;
                case "classnumber":
                    defect.ClassNumber = int.Parse(values[i]);
                    break;
            }
        }

        return defect;
    }

    private KlarfData RetrieveKlarfDataFromDatabase(string waferID)
    {
        KlarfData klarfData = new KlarfData();
        using (var connection = new SQLiteConnection(connectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                command.CommandText = "SELECT * FROM KlarfData WHERE WaferID = @WaferID";
                command.Parameters.AddWithValue("@WaferID", waferID);
                using (var reader = command.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        klarfData.WaferID = reader["WaferID"].ToString();
                        klarfData.FileTimestamp = reader["FileTimestamp"].ToString();
                        klarfData.InspectionStationID = reader["InspectionStationID"].ToString();
                        klarfData.SampleSize = Convert.ToInt32(reader["SampleSize"]);
                        klarfData.LotID = reader["LotID"].ToString();
                        klarfData.DeviceID = reader["DeviceID"].ToString();
                        klarfData.SetupID = reader["SetupID"].ToString();
                        klarfData.StepID = reader["StepID"].ToString();
                        klarfData.SampleOrientationMarkType = reader["SampleOrientationMarkType"].ToString();
                    }
                }
            }
        }
        return klarfData;
    }

    private List<DefectData> RetrieveDefectsFromDatabase(string waferID)
    {
        List<DefectData> defects = new List<DefectData>();
        using (var connection = new SQLiteConnection(connectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                command.CommandText = "SELECT * FROM Defects WHERE WaferID = @WaferID";
                command.Parameters.AddWithValue("@WaferID", waferID);
                using (var reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        defects.Add(new DefectData
                        {
                            X = Convert.ToSingle(reader["X"]),
                            Y = Convert.ToSingle(reader["Y"]),
                            Size = Convert.ToSingle(reader["Size"]),
                            ClassNumber = Convert.ToInt32(reader["ClassNumber"])
                        });
                    }
                }
            }
        }
        return defects;
    }

    private void DrawOrientationMark(Graphics g, string orientationMarkType, int imageSize)
    {
        float markSize = imageSize * 0.05f;
        float markPosition = imageSize * 0.95f;

        switch (orientationMarkType.ToLower())
        {
            case "notch":
                g.FillPie(Brushes.Black, markPosition - markSize, markPosition - markSize, markSize * 2, markSize * 2, 45, 90);
                break;
            case "flat":
                g.DrawLine(new Pen(Color.Black, 3), markPosition - markSize, markPosition, markPosition + markSize, markPosition);
                break;
        }
    }

    private Color GetDefectColor(int classNumber)
    {
        switch (classNumber % 5)
        {
            case 0: return Color.Red;
            case 1: return Color.Blue;
            case 2: return Color.Green;
            case 3: return Color.Orange;
            case 4: return Color.Purple;
            default: return Color.Black;
        }
    }

    private void DrawMetadata(Graphics g, KlarfData klarfData, int defectCount)
    {
        Font font = new Font("Arial", 10);
        int y = 10;

        DrawMetadataItem(g, font, "Wafer ID", klarfData.WaferID, ref y);
        DrawMetadataItem(g, font, "Lot ID", klarfData.LotID, ref y);
        DrawMetadataItem(g, font, "Device ID", klarfData.DeviceID, ref y);
        DrawMetadataItem(g, font, "Step ID", klarfData.StepID, ref y);
        DrawMetadataItem(g, font, "Total Defects", defectCount.ToString(), ref y);
    }

    private void DrawMetadataItem(Graphics g, Font font, string label, string value, ref int y)
    {
        g.DrawString($"{label}: {value}", font, Brushes.Black, new PointF(10, y));
        y += 20;
    }
}

public class KlarfData
{
    public string WaferID { get; set; }
    public string FileTimestamp { get; set; }
    public string InspectionStationID { get; set; }
    public int SampleSize { get; set; }
    public string LotID { get; set; }
    public string DeviceID { get; set; }
    public string SetupID { get; set; }
    public string StepID { get
```​​​​​​​​​​​​​​​​