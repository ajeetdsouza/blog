---
title: "Type-Safe Raytracing in Modern C++"
date: 2019-12-13T06:26:33+05:30
description: "Leveraging the C++ type system for safer and more robust code"
type: "post"
---

C++ has a very powerful type system. More often than not, however, this type system goes underutilized, leading to error-prone code and preventable bugs. Of late, I've been working through Peter Shirley's book, [*Raytracing in One Weekend*](https://raytracing.github.io/books/RayTracingInOneWeekend.html), and as I was following along, I ran into subtle issues that I felt could have been caught at compile time, had I used C++'s type system more effectively. Today, we will try to design types that make our code safer by preventing a number of such errors - at no runtime cost.

## The basic idea

The most fundamental data structure of a raytracer is the humble 3D vector. The book defines it as follows:

```c++
class vec3 {
public:
  vec3() {}
  vec3(float e0, float e1, float e2) { e[0] = e0; e[1] = e1; e[2] = e2; }
  inline float x() const { return e[0]; }
  inline float y() const { return e[1]; }
  inline float z() const { return e[2]; }
  inline float r() const { return e[0]; }
  inline float g() const { return e[1]; }
  inline float b() const { return e[2]; }

  inline const vec3& operator+() const { return *this; }
  inline vec3 operator-() const { return vec3(-e[0], -e[1], -e[2]); }
  inline float operator[](int i) const { return e[i]; }
  inline float& operator[](int i) { return e[i]; }

  inline vec3& operator+=(const vec3 &v2);
  inline vec3& operator-=(const vec3 &v2);
  inline vec3& operator*=(const vec3 &v2);
  inline vec3& operator/=(const vec3 &v2);
  inline vec3& operator*=(const float t);
  inline vec3& operator/=(const float t);

  inline float length() const { return sqrt(e[0]*e[0] + e[1]*e[1] + e[2]*e[2]); }
  inline float squared_length() const { return e[0]*e[0] + e[1]*e[1] + e[2]*e[2]; }
  inline void make_unit_vector();

  float e[3];
};
```

This allows us to reuse the `vec3` type for all vectors and colors, and defines every operator one might need. For a book on raytracing, this is convenient, since it keeps the code compact and simple. However, it also leads to code that is more error-prone, especially for larger projects.

## A bit of theory

Let's analyze the case of a Euclidean vector. [Wikipedia](https://en.wikipedia.org/wiki/Euclidean_vector) tells us that a Euclidean vector is a geometric object that has magnitude. It has 2 types:

- A **bound vector** has a fixed starting and ending point. They represent a _fixed point in space_, relative to some frame of reference. If we fix the starting point at the origin, this bound vector can represent the Cartesian coordinate of a point relative to the origin.
- A **free vector** has no initial point - it only has _direction_ and _magnitude_.

Often, however, we need a way to represent _only_ direction. A **unit vector** does exactly this - it is a free vector with its magnitude normalized to `1.0`.

Given the following data types, we can define some arithmetic operators on them:

```
BoundVec3 + FreeVec3 = BoundVec3
BoundVec3 - FreeVec3 = BoundVec3

BoundVec3 - BoundVec3 = BoundVec3

FreeVec3 + FreeVec3 = FreeVec3
FreeVec3 - FreeVec3 = FreeVec3

FreeVec3 * scalar = FreeVec3
FreeVec3 / scalar = FreeVec3

dot(FreeVec3, FreeVec3) = scalar
cross(FreeVec3, FreeVec3) = FreeVec3

UnitVec3 * scalar = FreeVec3
UnitVec3 / scalar = FreeVec3
```

## The code

For our data structures, we will make use of `double` rather than `float`. However, we want to be able to switch between them easily if needed, so we create a type alias:

```c++
using value_type = double;
```

Let's define a Euclidean vector and its two subtypes. As per the definition, it has just one property - its `length()`.

```c++
struct Vec3 {
 private:
  value_type _x;
  value_type _y;
  value_type _z;

 public:
  constexpr Vec3(const value_type x, const value_type y, const value_type z)
      : _x{x}, _y{y}, _z{z} {}

  constexpr value_type x() const { return this->_x; }
  constexpr value_type y() const { return this->_y; }
  constexpr value_type z() const { return this->_z; }

  constexpr value_type& x() { return this->_x; }
  constexpr value_type& y() { return this->_y; }
  constexpr value_type& z() { return this->_z; }

  value_type length() const {
    return std::hypot(this->x(), this->y(), this->z());
  }
};
```

Here, we use [const-overloading](https://isocpp.org/wiki/faq/const-correctness#const-overloading) to create inspector and mutator methods for `x`, `y`, and `z`. We will see why later.

The first subtype is a bound vector:

```c++
struct BoundVec3 : Vec3 {
  using Vec3::Vec3;

  constexpr explicit BoundVec3(const Vec3& vec3) : Vec3::Vec3{vec3} {}

  constexpr BoundVec3& operator+=(const FreeVec3& other) {
    this->x() += other.x();
    this->y() += other.y();
    this->z() += other.z();
    return *this;
  }

  constexpr BoundVec3& operator-=(const FreeVec3& other) {
    return *this += (-other);
  }
};

constexpr FreeVec3 operator-(const BoundVec3& v1, const BoundVec3& v2) {
  return FreeVec3{v1.x() - v2.x(), v1.y() - v2.y(), v1.z() - v2.z()};
}

constexpr BoundVec3 operator+(BoundVec3 v1, const FreeVec3& v2) {
  return v1 += v2;
}

constexpr BoundVec3 operator-(BoundVec3 v1, const FreeVec3& v2) {
  return v1 -= v2;
}
```

The second subtype is a free vector:

```c++
struct FreeVec3 : Vec3 {
  using Vec3::Vec3;

  constexpr explicit FreeVec3(const Vec3& vec3) : Vec3::Vec3{vec3} {}

  constexpr value_type dot(const FreeVec3& other) const {
    return this->x() * other.x() + this->y() * other.y() + this->z() * other.z();
  }

  constexpr FreeVec3 cross(const FreeVec3& other) const {
    return FreeVec3{this->y() * other.z() - this->z() * other.y(),
                    this->z() * other.x() - this->x() * other.z(),
                    this->x() * other.y() - this->y() * other.x()};
  }

  constexpr FreeVec3& operator+=(const FreeVec3& other) {
    this->x() += other.x();
    this->y() += other.y();
    this->z() += other.z();
    return *this;
  }

  constexpr FreeVec3& operator-=(const FreeVec3& other) {
    return *this += (-other);
  }

  constexpr FreeVec3& operator*=(const value_type scalar) {
    this->x() *= scalar;
    this->y() *= scalar;
    this->z() *= scalar;
    return *this;
  }

  constexpr FreeVec3& operator/=(const value_type scalar) {
    this->x() /= scalar;
    this->y() /= scalar;
    this->z() /= scalar;
    return *this;
  }
};

constexpr FreeVec3 operator+(const FreeVec3& v) { return v; }

constexpr FreeVec3 operator-(const FreeVec3& v) {
  return FreeVec3{-v.x(), -v.y(), -v.z()};
}

constexpr FreeVec3 operator+(FreeVec3 v1, const FreeVec3& v2) {
  return v1 += v2;
}

constexpr FreeVec3 operator-(FreeVec3 v1, const FreeVec3& v2) {
  return v1 -= v2;
}

constexpr FreeVec3 operator*(FreeVec3 v, const value_type scalar) {
  return v *= scalar;
}

constexpr FreeVec3 operator/(FreeVec3 v, const value_type scalar) {
  return v /= scalar;
}
```

Now, onto unit vectors. These are a little different, in the sense that they're not *truly* Euclidean vectors. Rather, they are an abstraction over free vectors that guarantee a length of `1.0`.[^1] Hence, there is no need of a `length()` function, and we want to prevent the user from mutating any fields in order to preserve its guarantees. In such a case, it may be more idiomatic to use [composition rather than inheritance](https://google.github.io/styleguide/cppguide.html#Inheritance):

```c++
struct UnitVec3 {
  UnitVec3(value_type x, value_type y, value_type z)
      : UnitVec3{FreeVec3{x, y, z}} {}

  explicit UnitVec3(const Vec3& vec3) : UnitVec3{FreeVec3{vec3}} {}

  explicit UnitVec3(const FreeVec3& vec3) : inner{vec3 / vec3.length()} {}

  constexpr value_type x() const { return this->to_free().x(); }
  constexpr value_type y() const { return this->to_free().y(); }
  constexpr value_type z() const { return this->to_free().z(); }

  constexpr const FreeVec3& to_free() const { return inner; }

 private:
  FreeVec3 inner;
};

constexpr FreeVec3 operator*(const UnitVec3& v, const value_type scalar) {
  return v.to_free() * scalar;
}

constexpr FreeVec3 operator/(const UnitVec3& v, const value_type scalar) {
  return v.to_free() / scalar;
}
```

As a bonus, composition also provides us cheap interconversion between the `UnitVec3` and `FreeVec3` types. While we have defined an explicit `to_free()` function here, we might instead allow implicit conversions via a [typecast operator](https://en.cppreference.com/w/cpp/language/cast_operator).[^2]

Also notice how `UnitVec3` only defines inspector methods for `x()`, `y()`, and `z()`. This ensures that the fields cannot be mutated, preserving our guarantees while keeping the API consistent.[^3]

Designing a `Color3` struct while keeping in mind the required guarantees should be fairly straightforward now, and is left as an exercise to the reader.

## Conclusion

Besides improving safety, better types enforce certain contracts, resulting in much clearer code. For example, a `Ray` implementation would go from this:

```c++
struct Ray {
  Vec3 origin;
  Vec3 direction;

  constexpr Vec3 at(const value_type t) const {
    return this->origin + (this->direction * t);
  }
};
```

to this:

```c++
struct Ray {
  BoundVec3 origin;
  UnitVec3 direction;

  constexpr BoundVec3 at(const value_type t) const {
    return this->origin + (this->direction * t);
    //                    ^^^^^^^^^^^^^^^^^^^^^ this becomes a FreeVec3
  }
};
```

Notice how our richer types not only model the problem more accurately, but also serve as a form of documentation.

Raytracers in production are heavily performance-oriented, and their design tends to lend itself to that requirement. While this article certainly does not intend to lay a foundation for any production-grade raytracer, I hope it provides some insight into how to design types and APIs in C++ that allow for code that is error-free, safer, and more robust.

If you have suggestions, questions, or complaints, feel free to [drop me a mail](mailto:98ajeet@gmail.com)!

[^1]: In our implementation, we will ignore the case where one or more values of the unit vector become `NaN`. One might handle this by raising an exception or by allowing the user to check manually via an `isnan()` function.

[^2]: A [converting constructor](https://en.cppreference.com/w/cpp/language/converting_constructor) would also have worked here, but ideally, `FreeVec3` should not know about `UnitVec3`, so this isn't as idiomatic.

[^3]: Another approach would be to make the member variables `x`, `y`, and `z` public, and use const member variables for `UnitVec3`, but that comes with [its own set of problems](https://www.drdobbs.com/the-problem-with-const-data-members/184403306).
