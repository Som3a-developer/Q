using System;
using System.IO;
using System.IO.Compression;
using System.Net.Sockets;
using System.Text;
using System.Xml.Serialization;

[Serializable]
public class FileMetaData
{
    public string FileName { get; set; }
    public long Size { get; set; }
    public string Extension { get; set; }
}

class Program
{
    static TcpClient client;
    static NetworkStream stream;

    static string seat, name, dept, section;
    static string createdFile = "";
    static string zipPath = "";

    static void Main()
    {
        Console.OutputEncoding = Encoding.UTF8;

        Console.Write("Seat ID: ");
        seat = Console.ReadLine();

        Console.Write("Arabic Name: ");
        name = Console.ReadLine();

        Console.Write("Department: ");
        dept = Console.ReadLine();

        Console.Write("Section: ");
        section = Console.ReadLine();

        client = new TcpClient("127.0.0.1", 9080);
        stream = client.GetStream();

        Console.WriteLine("Connected.");

        while (true)
        {
            string msg = ReceiveText();

            if (msg == null) break;

            Console.WriteLine("SERVER: " + msg);

            Handle(msg);
        }

        client.Close();
    }

    static void Handle(string msg)
    {
        if (msg.Contains("Send SeatID"))
        {
            SendText($"{seat}|{name}|{dept}|{section}");
        }
        

        else if (msg.Contains("job1 done"))
        {
            SendText("#ready2");
            ReceiveImage();
        }

        else if (msg.Contains("job2 done"))
        {
            SendText("#ready3");
        }

        else if (msg.Contains(".txt|"))
        {
            CreateFile(msg);
            SendText("#ready4");
        }

        else if (msg.Contains("Compress file"))
        {
            CompressFile();
            SendText("ZIP|" + zipPath);
        }

        else if (msg.Contains("job4 done"))
        {
            SendText("#ready5");
            ReceiveBinaryObject();
            SendText("#ready6");
            ReceiveXMLObject();
            SendText("#ready7");
        }

        else if (msg.Contains("send file"))
        {
            UploadFile(createdFile);
        }

        else if (msg.Contains("job7 done"))
        {
            Console.WriteLine("ALL JOBS DONE");
            Environment.Exit(0);
        }
    }

    static void SendText(string msg)
    {
        byte[] data = Encoding.UTF8.GetBytes(msg);
        stream.Write(data, 0, data.Length);
        Console.WriteLine("CLIENT: " + msg);
    }

    static string ReceiveText()
    {
        byte[] buffer = new byte[1024 * 1024];
        int read = stream.Read(buffer, 0, buffer.Length);

        if (read <= 0) return null;

        return Encoding.UTF8.GetString(buffer, 0, read);
    }

    static void ReceiveImage()
    {
        byte[] sizeBytes = new byte[4];
        stream.Read(sizeBytes, 0, 4);

        int size = BitConverter.ToInt32(sizeBytes, 0);

        byte[] image = new byte[size];

        int total = 0;

        while (total < size)
        {
            total += stream.Read(image, total, size - total);
        }

        string path = Path.Combine(
            Directory.GetCurrentDirectory(),
            "received.jpg");

        File.WriteAllBytes(path, image);

        Console.WriteLine("Image Saved: " + path);

        SendText(path);
    }

    static void CreateFile(string msg)
    {
        string[] p = msg.Split('|');

        createdFile = p[0];

        File.WriteAllText(createdFile, p[2]);

        Console.WriteLine("File Created: " + createdFile);
    }

    static void CompressFile()
    {
        zipPath = Path.ChangeExtension(createdFile, ".zip");

        if (File.Exists(zipPath))
            File.Delete(zipPath);

        using (ZipArchive zip = ZipFile.Open(zipPath, ZipArchiveMode.Create))
        {
            zip.CreateEntryFromFile(createdFile, Path.GetFileName(createdFile));
        }

        Console.WriteLine("ZIP Created: " + zipPath);
    }

    static void ReceiveBinaryObject()
    {
        BinaryReader br = new BinaryReader(stream);

        FileMetaData f = new FileMetaData
        {
            FileName = br.ReadString(),
            Size = br.ReadInt64(),
            Extension = br.ReadString()
        };

        Console.WriteLine("Binary Object:");
        Console.WriteLine(f.FileName);
        Console.WriteLine(f.Size);
        Console.WriteLine(f.Extension);
    }

    static void ReceiveXMLObject()
    {
        byte[] sizeBytes = new byte[4];
        stream.Read(sizeBytes, 0, 4);

        int size = BitConverter.ToInt32(sizeBytes, 0);

        byte[] data = new byte[size];

        int total = 0;

        while (total < size)
        {
            total += stream.Read(data, total, size - total);
        }

        XmlSerializer xs = new XmlSerializer(typeof(FileMetaData));

        using (MemoryStream ms = new MemoryStream(data))
        {
            FileMetaData f = (FileMetaData)xs.Deserialize(ms);

            Console.WriteLine("XML Object:");
            Console.WriteLine(f.FileName);
            Console.WriteLine(f.Size);
            Console.WriteLine(f.Extension);
        }
    }

    static void UploadFile(string file)
    {
        byte[] fileBytes = File.ReadAllBytes(file);

        byte[] nameBytes = Encoding.UTF8.GetBytes(Path.GetFileName(file));
        byte[] nameSize = BitConverter.GetBytes(nameBytes.Length);

        byte[] fileSize = BitConverter.GetBytes((long)fileBytes.Length);

        stream.Write(nameSize, 0, 4);
        stream.Write(nameBytes, 0, nameBytes.Length);

        stream.Write(fileSize, 0, 8);
        stream.Write(fileBytes, 0, fileBytes.Length);

        Console.WriteLine("Uploaded.");
    }
}
