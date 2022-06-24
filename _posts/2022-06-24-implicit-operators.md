---
layout: post
title: Implicit Operators
---

Implicit Operators in C# allow a class to implicitly convert to another class or type. In the example below, we can take a string like "local/TPALARM_v3" and cast it as an AlarmDesignation. This then gives us the ability to use all of our methods defined on this class.

```c#
namespace Honeywell.APMINDMSFT.ApmDataIngestLib
{
    using System;
    using System.ComponentModel;
    using System.Diagnostics.CodeAnalysis;

    [TypeConverter(typeof(AlarmDesignationConverter))]
    public class AlarmDesignation : IEquatable<AlarmDesignation>
    {
        private string type;
        private string scope;
        private int version;

        public AlarmDesignation(string raw)
        {
            this.Scoped = raw;
        }

        public bool HasType => !string.IsNullOrEmpty(this.type);
        public bool HasVersion => this.version > 0;
        public bool HasScope => !string.IsNullOrEmpty(this.scope);

        public int Version => this.version;
        public string Scope => this.scope;

        public static implicit operator string(AlarmDesignation value)
        {
            return value.Scoped;
        }

        public static implicit operator AlarmDesignation(string value)
        {
            return new AlarmDesignation(value);
        }

        public override string ToString()
        {
            return this.Scoped;
        }

        // ex. TPALARM
        public string Type
        {
            get => this.type;
            private set
            {
                if (!value.Contains("/") && !value.Contains("\\") && !value.Contains("_"))
                {
                    this.type = value;
                }
            }
        }

        // ex. TPALARM_v3
        public string Versioned
        {
            get => this.HasVersion ? $"{this.type}_v{this.version}" : this.type;
            private set
            {
                var parts = value.Split("_", 2);
                if (parts.Length == 1)
                {
                    this.Type = value;
                }
                else if (int.TryParse(parts[1][1..], out var version))
                {
                    this.Type = parts[0];
                    if (version > 0)
                    {
                        this.version = version;
                    }
                }
                else
                {
                    this.Type = parts[0];
                }
            }
        }

        // ex. local/TPALARM_v3
        public string Scoped
        {
            get => this.HasScope ? $"{this.scope}/{this.Versioned}" : this.Versioned;
            private set
            {
                var parts = value.Split("/", 2);
                if (parts.Length == 1)
                {
                    this.Versioned = value;
                }
                else if (parts[0].Equals("local", StringComparison.InvariantCultureIgnoreCase) || parts[0].Equals("global", StringComparison.InvariantCultureIgnoreCase))
                {
                    this.scope = parts[0].ToLower();
                    this.Versioned = parts[1];
                }
                else
                {
                    this.Versioned = parts[1];
                }
            }
        }
    }
}
```

To use this, we might...

```c#
AlarmDesignation ad = "global/TPALARM_v3";
```

This unit test passes as the object is implicitly converted to a string...

```c#
[Fact]
public void AlarmDesignation_AndString_AreEqual()
{
    AlarmDesignation x = "TPALARM_v3";
    string y = "TPALARM_v3";
    Assert.Equal(x, y);
}
```

But we still need the ToString() for something like this...

```c#
string val = $"Alarm Designation is: {ad}";
```

To use in a controller, like this...

```c#
[HttpGet("global/alarm-types/{alarmType}/designations")]
public ActionResult<AlarmDesignations> GetGlobalDesignationsForAlarmType(AlarmDesignation alarmType)
{
    return this.GetDesignationsByAlarmType("global", alarmType);
}
```

...we also need to provide a type converter like this...

```c#
using System;
using System.ComponentModel;
using System.Globalization;

public class AlarmDesignationConverter : TypeConverter
{
    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
    {
        return sourceType == typeof(string) || base.CanConvertFrom(context, sourceType);
    }

    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        if (value is string strval) return new AlarmDesignation(strval);
        return base.ConvertFrom(context, culture, value);
    }
}
```
