# Golang <img  width= "40px" src="https://user-images.githubusercontent.com/68103697/142988889-eb5c8ecf-7ae6-481b-8e8d-ffcf043830aa.png"/> 

# Golang
## 1. Introduction
<img  width= "100%" align="left" src="https://user-images.githubusercontent.com/68103697/143036012-ee9538a1-9a6d-4abe-ab16-7115a40d5613.png"/>    
    
## 2. Variables 
## 3. Primitive data
## 4. Constant enum
## 5. Array & Slice
## 6. Map & Construct
## 7. If - else 
## 8. Switch - case
## 9. Loop
## 10. Defer - Recover - Panic
## 11. Pointer
## 12. Function
## 13. Interface

## 14. Goroutine *(Chức năng xử lý song song)*
- Từ khóa _"go"_ là từ khóa khai báo tiến trình chạy song song:

```Go
package main

import (
	"fmt"
	"time"
)

func main() {
	go count("sheep") //khởi tạo goroutine khác
	count("Fish") //goroutine của hàm main
}

func count(name string) {
	for i := 1; i <= 5; i++ {
		fmt.Println(name, i)
		time.Sleep(time.Second)
	}
}
```
- Sau khi chạy chương trình, kết quả là:
```bash
Fish 1
sheep 1
sheep 2
Fish 2
Fish 3
sheep 3
sheep 4
Fish 4
Fish 5
sheep 5
```
> **Lưu ý:** Sau khi kết thúc goroutine của hàm main thì kết thúc chương trình 
 
 ### **Sử dụng WaitGroup**: *dùng để chạy song song nhiều goRoutine*
 ```Go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg = sync.WaitGroup{}
	wg.Add(2) // wg.Add(x) chờ x go routine
	go func() {
		count("sheep")
		wg.Done() //Phát tín hiệu đã hoàn tất
	}()

	go func() {
		count("Fish")
		wg.Done()
	}()
	wg.Wait() //Đợi cho tới khi nhận tín hiệu chạy hết goRoutine
	fmt.Println("Done")
}

func count(name string) {
	for i := 1; i <= 5; i++ {
		fmt.Println(name, i)
		time.Sleep(time.Second)
	}
}
 ```
### **Sừ dụng Mutex:** *Quản lý các tiến trình chạy song thực thi theo đúng thứ tự*
```Go
package main

import (
	"fmt"
	"sync"
)

var wg = sync.WaitGroup{}
var counter = 0
var m = sync.RWMutex{}

func main() {
	for i := 0; i < 10; i++ {
		wg.Add(2)
		m.RLock()
		go sayHello()
		m.Lock()
		go increment()
	}
	wg.Wait()
}

func sayHello() {
	fmt.Printf("Hello #%v\n", counter)
	m.RUnlock()
	wg.Done()
}

func increment() {
	counter++
	m.Unlock()
	wg.Done()
}
```

## 15. Channel *(Quản lý tài nguyên trong goRoutine)*
```Go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg = sync.WaitGroup{}
	wg.Add(2)
	ch := make(chan int, 50) //Tạo mới 1 channel kiểu int, có thể chứa tối đa 50 giá trị
	//Khai báo hàm lấy Channel
	go func(ch <-chan int) {
		i := <-ch //Lấy giá trị từ channel
		fmt.Println(i)
		wg.Done()
	}(ch)
	//Khai báo hàm nhận Channel
	go func(ch chan<- int) {
		ch <- 42 //Đưa giá trị vào trong Channel
		wg.Done()
	}(ch)
	wg.Wait()
}
```
- Kết quả sau khi chạy chương trình:
```42```

    - Biến ch lấy giá trị từ Channel, ***goRoutine thứ nhất*** chưa có giá trị
    - Sau đó, ch nhận giá trị từ ***goRoutine thứ hai*** truyền vào và in ra màn hình

### **Cách lấy và truyền nhiều Channel:**
```Go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg = sync.WaitGroup{}

	wg.Add(2)
	ch := make(chan int, 50)

	go func(ch <-chan int) {
		// Lấy nhiều Channel cùng 1 lúc
		for i := range ch {
			fmt.Println(i)
		}
		wg.Done()
	}(ch)

	go func(ch chan<- int) {
		ch <- 42
		ch <- 42
		close(ch) //đóng Channel
		wg.Done()
	}(ch)
	wg.Wait()
}
```
hoặc có thể thay vòng lặp for bằng đoạn mã sau:
```Go
	for {
			if i, isOK := <- ch; isOK {
				fmt.Println(i)
			} else {
				break
			}
		}
```
- ### **Từ khóa Select:** *Chọn giá trị lấy ra từ Channel*
```Go
	for {
			select {
				case i := <-ch1:
					fmt.Printf("Channel 1: %v\n", i)
				case j := <-ch2:
					fmt.Printf("Channel 2: %v\n", j)
				default:
					break
			}
		}
```
## 16. Pipeline fan-in fan-out *(Giải bài toán bằng cách chia nhỏ và chạy song song)*
- **Bài toán:** _Cho một mảng có n phần tử, tính tổng của bình phương các giá trị của phần tử trong mảng._
```Go
package main

import "fmt"

func main() {
	randomNumbers := []int{}
	for i := 1; i <= 1000; i++ {
		randomNumbers = append(randomNumbers, i)
	}
	//generate Pipeline object
	inputChan := generatePipeline(randomNumbers)

	//Fan-out
	c1 := fanOut(inputChan)
	c2 := fanOut(inputChan)
	c3 := fanOut(inputChan)
	c4 := fanOut(inputChan)

	//Fan-in
	c := fanIn(c1, c2, c3, c4)

	sum := 0
	for i := 0; i < len(randomNumbers); i++ {
		sum += <-c
	}
	fmt.Printf("Total sum of square %d", sum)
}

//Đẩy giá trị vào trong Channel
func generatePipeline(numbers []int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range numbers {
			out <- n
		}
		close(out)
	}()
	return out
}

//Tính và đưa giá trị ra thành nhiều Channel nhỏ hơn
func fanOut(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

//Tổng hợp các Channel thành 1 Channel duy nhất
func fanIn(inputChannel ...<-chan int) <-chan int {
	in := make(chan int)
	go func() {
		for _, c := range inputChannel {
			for n := range c {
				in <- n
			}
		}
	}()
	return in
}
```
## 17. Worker pool *(Chia bài toán ra nhiều tiến trình và cùng xử lý)*
**Bài toán:** *Cho 1 số nguyên n, in ra các số Fibonacci từ 1 đến n*
```Go
package main

import "fmt"

func main() {
	number := 100
	numberOfWorker := 5
	jobs := make(chan int, number)
	results := make(chan int, number)

	//Worker nhận dữ liệu
	for i := 0; i < numberOfWorker; i++ {
		go worker(jobs, results)
	}

	//Thêm các số vào Channel
	for i := 1; i <= number; i++ {
		jobs <- i
	}
	close(jobs)

	for j := 0; j < number; j++ {
		fmt.Println(<-results)
	}
}

//Hàm tạo worker
func worker(jobs <-chan int, results chan int) {
	for n := range jobs {
		results <- fib(n)
	}
}

//Hàm tính Fibonacci sử dụng đệ quy
func fib(n int) int {
	if n <= 1 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```   
# ***DESIGN PATTERN FOR*** *Golang*
## 18. Creational pattern - Singleton *- Sử dụng để tạo ra một đối tượng xuyên suốt cho một struct (Data, IO, Log)*
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143162544-1f108383-bd85-40e3-9b2d-61d001cd014b.png"/>   

**File** *singleton.go*
```Go
package singleton

//Singleton Interface
type Singleton interface {
	AddOne() int
}

type singleton struct {
	count int
}

var instance *singleton

func init() {
	instance = &singleton{count: 100}
}

//GetInstance function return object
func GetInstance() Singleton {
	return instance
}

func (s *singleton) AddOne() int {
	s.count++
	return s.count
}
```
**File** *main.go*
```Go
package main

import (
	"fmt"
	"singleton"
	"time"
)

func main() {
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Printf("&p\n", singleton.GetInstance())
		}()
	}
	time.Sleep(time.Second * 10)
}
```


## 19. Creational pattern - Builder 
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143161487-306b5a05-a052-4d18-a52f-f026f3192996.png"/>
<br>
</br>

- *dùng khi tạo 1 Struct có nhiều thuộc tính*
- *dùng khi muốn tạo ra 1 đối tượng cụ thể hơn cho struct*
- *tạo ra 1 struct mà không cần phải khai báo tất cả các thuộc tính của struct đó*
- Các thành phần của 1 builder: 
	- Product
	- Builder interface
	- Concrete Builder 1
	- Concrete Builder 2
	- Director

**File** *house.go*
```Go
package builder

//House is struct
type House struct {
	windowType string
	doorType   string
	floor      int
}

//GetWindowType is getter
func (h *House) GetWindowType() string {
	return h.windowType
}

//GetDoorType is getter
func (h *House) GetDoorType() string {
	return h.doorType
}

//GetNumFloor is getter
func (h *House) GetNumFloor() int {
	return h.floor
}
```
**File** *IBuilder.go*
```Go
package builder

//IBuilder is interface
type IBuilder interface {
	setWindowType()
	setDoorType()
	setNumFloor()
	getHouse() House
}

//GetBuilder is function
func GetBuilder(builderType string) IBuilder {
	switch builderType {
	case "normal":
		return &normalBuilder{}
	case "igloo":
		return &iglooBuilder{}
	}
}
```
**File** *normalBuilder.go*
```Go
package builders

type normalBuilder struct {
	windowType string
	doorType   string
	floor      int
}

func newNormalBuilder() *normalBuilder {
	return &normalBuilder{}
}

func (b *normalBuilder) setWindowType() {
	b.windowType = "Wooden Window"

}

func (b *normalBuilder) setDoorType() {
	b.doorType = "Wooden Door"
}

func (b *normalBuilder) setNumFloor() {
	b.floor = 2
}

func (b *normalBuilder) getHouse() {
	return House{
		doorType:   b.doorType,
		windowType: b.windowType,
		floor:      b.floor,
	}
}
```
**File** *iglooBuider.go*
```Go
package builder

type iglooBuilder struct {
	windowType string
	doorType   string
	floor      int
}

func newIgloolBuilder() *iglooBuilder {
	return &iglooBuilder{}
}

func (b *iglooBuilder) setWindowType() {
	b.windowType = "Snow Window"
}

func (b *iglooBuilder) setDoorType() {
	b.doorType = "Snow Door"
}

func (b *iglooBuilder) setNumFloor() {
	b.floor = 1
}

func (b *iglooBuilder) getHouse() {
	return House{
		doorType:   b.doorType,
		windowType: b.windowType,
		floor:      b.floor,
	}
}
```
**File** *director.go*
```Go
package builder.Builder

//Director is struct
type Director struct {
	builder IBuilder
}

//NewDirector is function
func NewDirector(b IBuilder) *Director {
	return &Director{
		buider : b,
	}
}

//SetBuider is function
func (d *Director) SetBuider(b IBuilder) {
	d.buider = b
}

//BuildHouse id function
func (d *Director) BuildHouse() House {
	d.builder.setDoorType()
	d.builder.setNumFloor()
	d.builder.setWindowType()
	return d.buider.getHouse()
}
```
**File** *main.go*
```Go
package main

import (
	"buider"
	"fmt"
)

func main() {
	normalBuilder := builder.GetBuilder("normal")
	iglooBuilder := builder.GetBuilder("igloo")

	director := buider.NewDirector(normalBuilder)
	normalHouse := director.BuildHouse()

	fmt.Printf("Normal House Door Type: %s\n", normalHouse.GetDoorType())
	fmt.Printf("Normal House Window Type: %s\n", normalHouse.GetWindowType())
	fmt.Printf("Normal House Num Floor: %s\n", normalHouse.GetNumFloor())

	director.SetBuilder(iglooBuider)
	iglooHouse := director.BuildHouse()

	fmt.Printf("Igloo House Door Type: %s\n", iglooHouse.GetDoorType())
	fmt.Printf("Igloo House Window Type: %s\n", iglooHouse.GetWindowType())
	fmt.Printf("Igloo House Num Floor: %s\n", iglooHouse.GetNumFloor())
}

```
## 20. Creational pattern - Abstract factory
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143162802-85350dbe-580a-467d-a3a5-1460902686a9.png"/>   


- Abstract Product Interface 1
- Abstract Product Interface 2
- Concrete Product 1
- Concrete Product 2
- Abstract Factory Interface
- Concrete Factory 1
- Concrete Factory 2

### **Các file:**
**File** *iShoe.go*
```Go
package abstractfactory

//IShoe is interface
type IShoe interface {
	setLogo(logo string)
	setSize(size int)
	GetLogo() string
	GetSize() int
}

type shoe struct {
	logo string
	size int
}

func (s *shoe) setLogo(logo string) {
	s.logo = logo
}

func (s *shoe) setSize(size int) {
	s.size = size
}

func (s *shoe) GetLogo() string {
	return s.logo
}

func (s *shoe) GetSize() int {
	return s.size
}
```
**File** *iShort.go*
```Go
package abstractfactory

//IShort is interface
type IShort interface {
	setLogo(logo string)
	setSize(size int)
	GetLogo() string
	GetSize() int
}

type short struct {
	logo string
	size int
}

func (s *short) setLogo(logo string) {
	s.logo = logo
}

func (s *short) setSize(size int) {
	s.size = size
}

func (s *short) GetLogo() string {
	return s.logo
}

func (s *short) GetSize() int {
	return s.size
}
```
**File** *adidasShoe.go*
```Go
package abstractfactory

type adidasShoe struct {
	shoe
}
```
**File** *adidasShort.go*
```Go
package abstractfactory

type adidasShort struct {
	short
}
```
**File** *nikeShoe.go*
```Go
package abstractfactory

type nikeshoe struct {
	shoe
}
```
**File** *nikeShort.go*
```Go
package abstractfactory

type nikeShort struct {
	short
}
```
**File** *iSportFactory.go*
```Go
package abstractfactory

//ISportFactory is interface
type ISportFactory interface {
	MakeShoe() IShoe
	MakeShort() IShort
}

//GetSportFactory is function
func GetSportsFactory(brand string) ISportFactory {
	switch brand {
	case "adidas":
		return &Adidas
	case "nike":
		return &Nike
	}
	return nil
}
```
**File** *adidas.go*
```Go
package abstractfactory

type Adidas struct{}

func (a *Adidas) MakeShoe() IShoe {
	return &adidasShoe{
		shoe: shoe{
			logo: "adidas",
			size: 14,
		},
	}
}

func (a *Adidas) MakeShort() IShort {
	return &adidasShort{
		short: short{
			logo: "adidas",
			size: 14,
		},
	}
}
```
**File** *nike.go*
```Go
package abstractfactory

type Nike struct{}

func (n *Nike) MakeShoe() IShoe {
	return &nikeShoe{
		shoe: shoe{
			logo: "Nike",
			size: 12,
		},
	}
}

func (n *Nike) MakeShort() IShort {
	return &nikeShort{
		short: short{
			logo: "Nike",
			size: 12,
		},
	}
}
```
**File** *main.go*
```Go
package main

import (
	"abstractfactory"
	"fmt"
)

func main() {
	adidasFactory := abstractfactory.GetSportFactory("adidas")
	adidasShoe := adidasFactory.MakeShoe()
	printShoeDetails(adidasShoe)
	adidasShort := adidasFactory.MakeShort()
	printShortDetails(adidasShort)

	nikeFactory := abstractfactory.GetSportFactory("Nike")
	nikeShoe := nikeFactory.MakeShoe()
	printShoeDetails(nikeShoe)
	nikeShort := nikeFactory.MakeShort()
	printShortDetails(nikeShort)

}

func printShoeDetails(s abstractfactory.IShoe) {
	fmt.Printf("Logo: %s\n", s.GetLogo())
	fmt.Printf("Size: %d\n", s.GetSize())
}

func printShortDetails(s abstractfactory.IShort) {
	fmt.Printf("Logo: %s\n", s.GetLogo())
	fmt.Printf("Size: %d\n", s.GetSize())
}
```
## 21. Creational pattern - Prototype
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143162956-6861542f-7e02-42d3-b5d4-95157ac4ab4c.png"/>
- **When use ?**

  - *Sử dụng khi muốn tạo ra 1 bản sao của đối tượng mà struct của nó bên trong chứa thuộc tính của 1 struct khác*

- Các thành phẩn của Prototype:

	- Prototype Interface

		- Phương thức Clone()

	- Concrete Prototype 1, 2,...

**File** *INode.go*
```Go
//File INode.go
package prototype

import "fmt"

type INode interface {
	Clone() INode
	Print(s string)
}
```
**File** *File.go*
```Go
//File File.go
package prototype

import "fmt"

type File struct {
	Name string
}

func Print(f *file) Print(s string) {
	fmt.Println(s + f.Name)
}

func (f*file) Clone() INode {
	return &File(name : f.Name + "_Clone")
}
```
**File** *Folder.go*
```Go
//File Folder.go
package prototype

import "fmt"

type Folder struct {
	Childrens []INode
	Name string
}

func (f *Folder) Print(s string) {
	fmt.Println(s + f.Name)
	for _, i := range f.Childrens {
		i.Print(s + s)
	}
}

func (f *Clone) Clone() INode {
	cloneFolder := &Folder{Name : f.Name + "_Clone"}
	var tempChildrens []INode
	for _, i := range f.Childrens {
		copy := i.Clone()
		tempChildrens = append(tempChildrens, copy)
	}
	cloneFolder.Childrens = tempChildrens
	return cloneFolder
}
```
**File** *main.go*
```Go
//File main.go
package main

import (
	"fmt"
	p "prototype"
)

func main() {
	file1 := &p.file{Name : "File 1"}
	file2 := &p.file{Name : "File 2"}
	file3 := &p.file{Name : "File 3"}
	folder1 := &p.Folder{
		Childrens : []p.INode{file1},
		Name : "Folder 1"
	}

	folder2 := &p.Folder{
		Childrens : []p.INode{file1, file2, file3},
		Name : "Folder 2"
	}

	fmt.Println("\n Printing for folder 2")
	folder2.Print("   ")
	cloneFolder := folder2.Clone()
	fmt.Println("\n Printing for clone folder 2")
	folder2.Print("   ")
}
```   

## 22. Behavioral pattern - Chain of responsibility
**When use???**
- Khi ta có 1 đối tượng và đối tượng đó được xử lý bằng một chuỗi các hành vi xử lý   
   
**Các thành phần:**  

- Client
- Handler interface

	- Phương thức Execute()  
	- Phương thức setNext()  
- Concrete Handler ,...    
**File** *patient.go*
```Go
//File patient.go
package chainofresponsibility

import "fmt"

type Patient struct {
	Name string
	isRegistered bool
	isDoctorChecked bool
	isMedicineProvide bool
	isPaid bool
}
```   
**File** *department.go*
```Go
//File department.go
package chainofresponsibility

type Department interface {
	Execute(*Patient)
	SetNext(Department)
}
```
**File** *reception.go*
```Go
//File reception.go
package chainofresponsibility

import "fmt"

type Reception struct {
	next Department
}

func (r *Reception) Execute(p *Patient) {
	if p.isRegistered {
		fmt.Println("Patient registration has already done")
		r.next.Execute(p)
		return
	}
	fmt.Println("Reception registering patient")
	p.isRegistered = true
	r.next.Execute(p)
}

func (r *Reception) SetNext(next Department) {
	r.next = next
}
```   
**File** *doctor.go*
```Go
//File doctor.go
package chainofresponsibility

import "fmt"

type Doctor struct {
	next Department
}

func (d *Doctor) Execute(p *Patient) {
	if p.isDoctorChecked {
		fmt.Println("Patient already checked by doctor")
		d.next.Execute(p)
		return
	}
	fmt.Println("Doctor is checking patient")
	p.isDoctorChecked = true
	d.next.Execute(p)
}

func (d *Doctor) SetNext(next Department) {
	d.next = next
}
```   
**File** *medical.go* 
```Go
//File medical.go
package chainofresponsibility

import "fmt"

type Medical struct {
	next Department
}

func (m *Medical) Execute(p *Patient) {
	if p.isMedicineProvide {
		fmt.Println("Patient already provide medicine")
		m.next.Execute(p)
		return
	}
	fmt.Println("We are providing medicine to patient")
	p.isMedicineProvide = true
	m.next.Execute(p)
}

func (m *Medical) SetNext(next Department) {
	m.next = next
}
```   
**File** *cashier.go*
```Go
//File cashier.go
package chainofresponsibility

import "fmt"

type Cashier struct {
	next Department
}

func (c *Cashier) Execute(p *Patient) {
	if p.isPaid {
		fmt.Println("Patient already paid their bill")
		r.next.Execute(p)
		return
	}
	fmt.Println("Patient is paying the bill")
	p.isPaid = true

}

func (c *Cashier) SetNext(next Department) {
	c.next = next
}
```   
**File** *main.go*   
```Go
//File main.go
package main

import (
	 c "chainofresponsibility"
)
func main() {
	cashier := &c.Cashier{}

	medical := &c.Medical{}
	medical.setNext(cashier)

	doctor := &c.Doctor{}
	doctor.setNext(medical)

	reception := &c.Reception{}
	doctor.setNext(doctor)

	patient := &c.Patient{Name: "Yuh"}
	reception.Execute(patient)
}
```   
## 23. Behavioral pattern - Command   
**When use**   


## 24. Behavioral pattern - Iterator
## 25. Behavioral pattern - Mediator
## 26. Behavioral pattern - Memento
## 27. Behavioral pattern - Observer
## 28. Behavioral pattern - State
## 29. Behavioral pattern - Strategy
## 30. Behavioral pattern - Template method
## 31. Behavioral pattern - Visitor
## 32. Structural pattern - Adapter
## 33. Structural pattern - Bridge
## 34. Structural pattern - Composite
## 35. Structural pattern - Facade
## 36. Structural pattern - Flyweight
## 37. Structural pattern - Proxy
## 38. Structural pattern - Decorator
## 39. Stack & Heap & Cấp phát bộ nhớ


