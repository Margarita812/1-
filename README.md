using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Text.RegularExpressions;

class LogAnalyzer
{
    static void Main(string[] args)
    {
        if (args.Length == 0)
        {
            Console.WriteLine("Укажите путь к файлу лога.");
            Console.WriteLine("Пример: LogAnalyzer.exe C:\\logs\\app.log");
            return;
        }

        string logFilePath = args[0];

        if (!File.Exists(logFilePath))
        {
            Console.WriteLine($"Файл не найден: {logFilePath}");
            return;
        }

        try
        {
            var result = ProcessLogFile(logFilePath);
            SaveAndPrintReport(result, logFilePath);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Ошибка при обработке: {ex.Message}");
        }
    }

    static LogReport ProcessLogFile(string logFilePath)
    {
        var errorMessages = new Dictionary<string, int>();
        var warnMessages = new Dictionary<string, int>();
        int totalErrors = 0;
        int totalWarns = 0;

        string[] lines = File.ReadAllLines(logFilePath);
        Regex logRegex = new Regex(@"^\[(.*?)\]\s+\[(ERROR|WARN|INFO|DEBUG)\]\s+(.*)$");

        foreach (string line in lines)
        {
            Match match = logRegex.Match(line);
            if (!match.Success) continue;

            string level = match.Groups[2].Value;
            string message = match.Groups[3].Value.Trim();

            if (level == "ERROR")
            {
                totalErrors++;
                if (errorMessages.ContainsKey(message))
                    errorMessages[message]++;
                else
                    errorMessages[message] = 1;
            }
            else if (level == "WARN")
            {
                totalWarns++;
                if (warnMessages.ContainsKey(message))
                    warnMessages[message]++;
                else
                    warnMessages[message] = 1;
            }
        }

        var topError = errorMessages.OrderByDescending(x => x.Value).FirstOrDefault();
        var topWarn = warnMessages.OrderByDescending(x => x.Value).FirstOrDefault();

        var topIssues = new List<TopIssue>();

        if (topError.Key != null)
        {
            topIssues.Add(new TopIssue
            {
                Message = topError.Key,
                Count = topError.Value,
                Level = "ERROR"
            });
        }

        if (topWarn.Key != null)
        {
            topIssues.Add(new TopIssue
            {
                Message = topWarn.Key,
                Count = topWarn.Value,
                Level = "WARN"
            });
        }

        return new LogReport
        {
            GeneratedAt = DateTime.Now.ToString("yyyy-MM-ddTHH:mm:ss"),
            TotalErrors = totalErrors,
            TotalWarns = totalWarns,
            TopIssues = topIssues
        };
    }

    static void SaveAndPrintReport(LogReport report, string logFilePath)
    {
        string directory = Path.GetDirectoryName(logFilePath);
        string reportPath = Path.Combine(directory, "report.json");

        var options = new JsonSerializerOptions { WriteIndented = true };
        string json = JsonSerializer.Serialize(report, options);

        File.WriteAllText(reportPath, json);
        Console.WriteLine("Содержимое report.json:");
        Console.WriteLine(json);
        Console.WriteLine($"\nОтчёт сохранён в: {reportPath}");
    }
}

class LogReport
{
    public string GeneratedAt { get; set; }
    public int TotalErrors { get; set; }
    public int TotalWarns { get; set; }
    public List<TopIssue> TopIssues { get; set; }
}

class TopIssue
{
    public string Message { get; set; }
    public int Count { get; set; }
    public string Level { get; set; }
}
