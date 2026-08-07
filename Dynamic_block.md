Absolutely. Since you've already learned **resource** and **data** blocks, the next important concept is the **dynamic block**. This is a bit more advanced, but it's extremely useful in real-world Terraform projects.

---

# What is a Dynamic Block?

A **dynamic block** is used to **generate one or more nested blocks automatically** inside a resource.

Instead of writing the same nested block multiple times, you let Terraform create them in a loop.

Think of it like this:

* **for_each** → Creates multiple **resources**
* **dynamic** → Creates multiple **nested blocks inside one resource**

---

# Why Do We Need It?

Suppose you're creating a Security Group.

Normally, if you want three inbound rules, you'd write:

```hcl
resource "aws_security_group" "web" {

  name = "web-sg"

  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 443
    to_port   = 443
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

}
```

Notice that you're repeating the `ingress` block.

If you had 50 ports, you'd write 50 `ingress` blocks.

That's not ideal.

---

# Solution: Dynamic Block

Instead:

```hcl
variable "ports" {
  default = [22, 80, 443]
}

resource "aws_security_group" "web" {

  name = "web-sg"

  dynamic "ingress" {

    for_each = var.ports

    content {

      from_port = ingress.value

      to_port = ingress.value

      protocol = "tcp"

      cidr_blocks = ["0.0.0.0/0"]

    }

  }

}
```

Terraform automatically creates:

```
ingress {
  from_port = 22
  ...
}

ingress {
  from_port = 80
  ...
}

ingress {
  from_port = 443
  ...
}
```

---

# Syntax

```hcl
dynamic "<BLOCK_NAME>" {

  for_each = ...

  iterator = ...

  content {

    ...

  }

}
```

---

## Understanding Each Part

### 1. dynamic

This tells Terraform:

> "Generate nested blocks."

Example:

```hcl
dynamic "ingress"
```

means

> Generate multiple `ingress` blocks.

---

### 2. for_each

Specifies how many nested blocks to create.

Example:

```hcl
for_each = [22,80,443]
```

Terraform loops over every value.

---

### 3. iterator (Optional)

If you don't specify an iterator, Terraform uses the block name.

Example:

```hcl
dynamic "ingress" {

  for_each = var.ports

  content {

      from_port = ingress.value

  }

}
```

Here

```
ingress.value
```

is the current value.

You can rename it:

```hcl
dynamic "ingress" {

  for_each = var.ports

  iterator = port

  content {

      from_port = port.value

  }

}
```

Both are valid.

---

### 4. content

Everything inside `content {}` becomes the generated nested block.

Example

```hcl
content {

   from_port = ingress.value

   to_port = ingress.value

}
```

Terraform copies this once for each item.

---

# Execution Flow

Suppose

```hcl
variable "ports" {

 default = [22,80,443]

}
```

Terraform reads

```
22

80

443
```

Loop 1

```
ingress {

from_port = 22

}
```

Loop 2

```
ingress {

from_port = 80

}
```

Loop 3

```
ingress {

from_port = 443

}
```

---

# Another Example

Suppose every rule has different CIDRs.

```hcl
variable "rules" {

 default = [

 {

 port = 22

 cidr = "10.0.0.0/16"

 },

 {

 port = 80

 cidr = "0.0.0.0/0"

 },

 {

 port = 443

 cidr = "0.0.0.0/0"

 }

 ]

}
```

Dynamic block

```hcl
resource "aws_security_group" "web" {

 dynamic "ingress" {

   for_each = var.rules

   content {

      from_port = ingress.value.port

      to_port = ingress.value.port

      protocol = "tcp"

      cidr_blocks = [ingress.value.cidr]

   }

 }

}
```

Terraform generates

```
ingress {

from_port = 22

cidr_blocks = ["10.0.0.0/16"]

}

ingress {

from_port = 80

cidr_blocks = ["0.0.0.0/0"]

}

ingress {

from_port = 443

cidr_blocks = ["0.0.0.0/0"]

}
```

---

# Real-Time DevOps Example

Suppose your DevOps team manages environments like this:

| Environment | Ports             |
| ----------- | ----------------- |
| Development | 22, 80            |
| Testing     | 22, 80, 443       |
| Production  | 22, 80, 443, 8080 |

Without a dynamic block, you'd need separate `ingress` definitions for each environment.

Instead:

```hcl
variable "ports" {

default = [22,80,443,8080]

}

resource "aws_security_group" "web" {

dynamic "ingress" {

for_each = var.ports

content {

from_port = ingress.value

to_port = ingress.value

protocol = "tcp"

cidr_blocks = ["0.0.0.0/0"]

}

}

}
```

Now, if the team wants to allow port `8443`, you only update the variable:

```hcl
ports = [22,80,443,8080,8443]
```

Terraform automatically generates the additional `ingress` block.

---

# Dynamic Block vs for_each

This is where many beginners get confused.

## `for_each` on a Resource

Creates **multiple resources**.

```hcl
resource "aws_instance" "server" {

for_each = {

web = "t2.micro"

app = "t3.micro"

}

instance_type = each.value

}
```

Terraform creates:

```
EC2 Web

EC2 App
```

Two separate EC2 instances.

---

## `dynamic`

Creates **multiple nested blocks inside a single resource**.

```hcl
resource "aws_security_group" "web" {

dynamic "ingress" {

...

}

}
```

Terraform creates:

```
ONE Security Group

↓

Many ingress blocks
```

Not multiple security groups.

---

# When Should You Use Dynamic Blocks?

Use a dynamic block when:

* A resource contains **repeatable nested blocks** (like `ingress`, `egress`, `ebs_block_device`, etc.).
* The number of nested blocks is **not fixed**.
* You want to avoid repeating the same configuration.

Avoid using a dynamic block if there is only one nested block, as it can make the code harder to read.

---

# Summary

| Concept    | Purpose                                              |
| ---------- | ---------------------------------------------------- |
| `resource` | Creates infrastructure                               |
| `data`     | Reads existing infrastructure                        |
| `for_each` | Creates multiple resources                           |
| `dynamic`  | Creates multiple nested blocks within one resource   |
| `content`  | Defines the template for each generated nested block |
| `iterator` | Represents the current item in the loop (optional)   |

## A simple way to remember

Imagine you're filling out forms:

* **`resource`** = Print a new form.
* **`for_each`** = Print 10 copies of the form.
* **`dynamic`** = Fill in 10 repeated sections inside **one** form.
* **`data`** = Read information from an existing form instead of creating a new one.

For AWS DevOps, the most common place you'll encounter dynamic blocks is when creating **Security Groups** with multiple ingress/egress rules, but they're also useful for resources like load balancers, launch templates, and IAM policies that have repeatable nested configuration blocks.
