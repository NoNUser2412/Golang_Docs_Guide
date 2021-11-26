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
![go](https://user-images.githubusercontent.com/68103697/143244421-1dea961c-8caf-4e41-82fe-95397df35fe7.png)

 
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
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143171894-6027328d-024c-49c1-99e3-af119da810b3.png"/>

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
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172000-48d0b93f-5013-4eb5-b9ec-93a572c65771.png"/>   

**Các thành phần:**  
- Involker  
- Command interface   
	- Phương thức Execute()    
   
- Concrete Command ,...    
- Receiver Interface    
- Concrete receiver    
    
**File:** *button.go*   
```Go
//File button.go
package commmand

import "fmt"

type Button struct {
	Command Command
}

func (b *Button) Press() {
	b.Command.execute()
}
```    

**File:** *command.go*   
```Go
//File command.go
package commmand

type Command interface {
	execute()
}
```  
**File:** *onCommand.go*   
```Go
//File onCommand.go
package commmand

//File onCommand.go
package commmand

type OnCommand struct {
	Device Device
}

func (c *OnCommand) execute() {
	c.Device.on()
}
```
**File:** *offCommand.go*   
```Go
//File offCommand.go
package commmand

type OffCommand struct {
	Device Device
}

func (c *OffCommand) execute() {
	c.Device.off()
}
```
**File:** *device.go*   
```Go
//File device.go
package commmand

type Device interface {
	on()
	off()
}
```
**File:** *tivi.go*   
```Go
//File tivi.go
package commmand

import "fmt"

type Tivi struct {
	isRunning bool
}

func (t *Tivi) on() {
	t.isRunning = true
	fmt.Println("Turning tivi on")
}

func (t *Tivi) off() {
	t.isRunning = false
	fmt.Println("Turning tivi off")
}
```  

**File:** *main.go*   
```Go
//File main.go
package main

import c "command"

func main() {
	tv := &c.Tivi{}
	onCommand := &c.OnCommand{
		Device: tv,
	}
	offCommand := &c.OffCommand{
		Device: tv,
	}

	onButton := &c.OnButton{
		Command: onCommand,
	}
	onButton.Press()

	offButton := &c.OffButton{
		Command: offCommand,
	}
	offButton.Press()
}
```   

## 24. Behavioral pattern - Iterator
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172134-b01e5e01-595f-43da-8a8f-13e40e98987e.png"/>    
**Các thành phần:**   
- Client   
- Collection interface   
- Concrete Collection 1   
- Iterator interface    
    
	- Phương thức hasNext()   
	- Phương thức getNext()   
- Concrete iterator       


**File** *user.go*
```Go
//File user.go
package iterator

import "fmt"

type User struct {
	Name string
	Age int
}
```
**File** *collection.go*
```Go
//File collection.go
package iterator

import "fmt"

type Collection interface {
	CreateIterator() Iterator
}
```   
**File** *userCollection.go*
```Go
//File userCollection.go
package iterator

import "fmt"

type UserCollection struct {
	user [] *User
}

func (u *UserCollection) CreateIterator() Iterator {
	return &UserIterator{
		users : u.users,
	}
}
```   
**File** *iterator.go*
```Go
//File iterator.go
package iterator

import "fmt"

type Iterator interface {
	HasNext() bool
	GetNext() *User
}
```   
**File** *userIterator.go*
```Go
//File userIterator.go
package iterator

import "fmt"

type UserIterator struct {
	users []*User
	index int
}

func (u *UserIterator) HasNext() bool {
	if u.index < len(u.users) {
		return true
	}
	return false
}

func (u *UserIterator) GetNext() *User {
	if u.HasNext() {
		user := u.users(u.index)
		u.index++
		return user
	}
	return nil
}
```    
**File** *main.go*
```Go
//File main.go
package main

import (
	"fmt"
	i "iterator"
)

func main() {
	user1 := &i.User {
		Name : "Yuh",
		Age : 18,
	}
	user2 := &i.User {
		Name : "Tom",
		Age : 40,
	}
	UserCollection := &UserCollection{
		Users : []*i.User{user1, user2},
	}
	iterator := userCollection.CreateIterator()

	for iterator.HasNext() {
		user := iterator.GetNext()
		fmt.Printf("User is %+v\n", user)
	}
}
``` 


## 25. Behavioral pattern - Mediator
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172523-f7696e35-76e2-4f4c-80c1-e5b097f62348.png"/>     
**Các thành phần**   
- Client Interface   
- Concrete client,...   
- Mediator interface   
- Concrete Mediator   
   
**File** *train.go*    
```Go
//File train.go
package meidator

import (
	"fmt"
	"sync"
)

type Train interface {
	RequestArrival()
	Departure()
	PermitArrival()
}
```  
**File** *mediator.go*    
```Go
//File mediator.go
package mediator

type Mediator interface {
	CanLand(Train) bool
	NotifyFree()
}
```   
**File** *passengerTrain.go*    
```Go
//File passengerTrain.go
package mediator

type PassengerTrain struct {
	Mediator Mediator
}

func (p *PassengerTrain) RequestArrival() {
	if p.Mediator.CanLand(p) {
		fmt.Println("Passenger Train: Landing")
	} else {
		fmt.Println("Passenger Train: Waiting")
	}
}

func (p *PassengerTrain) Departure() {
	fmt.Println("Passenger Train: Leaving")
	p.Mediator.NotifyFree()
}

func (p *PassengerTrain) PermitArrival() {
	fmt.Println("Passenger Train: Arrival Permitted. Landing")
}
```   
**File** *goodTrain.go*    
```Go
//File goodTrain.go
package mediator

type GoodTrain struct {
	Mediator Mediator
}

func (g *GoodTrain) RequestArrival() {
	if g.Mediator.CanLand(g) {
		fmt.Println("Good Train: Landing")
	} else {
		fmt.Println("Good Train: Waiting")
	}
}

func (g *GoodTrain) Departure() {
	fmt.Println("Good Train: Leaving")
	g.Mediator.NotifyFree()
}

func (g *GoodTrain) PermitArrival() {
	fmt.Println("Good Train: Arrival Permitted. Landing")
}
```    
**File** *stationManager.go*    
```Go
//File stationManager.go
package mediator

import "sync"

type StationManager struct {
	isPlatformFree bool
	lock *sync.Mutex
	trainQueue []Train
}

func NewStationManager() *StationManager {
	return &StationManager{
		isPlatformFree : true,
		lock : &sync.Mutex{},
	}
}

func (s *StationManager) CanLand(t Train) bool{
	s.lock.Lock()
	defer s.lock.Unlock()
	if s.isPlatformFree {
		s.isPlatformFree = false
		return true
	}
	s.trainQueue = append(s.trainQueue, t)
	return false
}

func (s *StationManager) NotifyFree() {
	s.lock.Lock()
	defer s.lock.Unlock()
	if !s.isPlatformFree {
		s.isPlatformFree = true
	}
	if len(s.trainQueue) > 0 {
		firstTrainInQueue := s.trainQueue[0]
		s.trainQueue := s.trainQueue[1:]
		firstTrainInQueue.PermitArrival()
	}
}
```   
**File** *main.go*    
```Go
//File main.go
package main

import m "mediator"

func main() {
	stationManager := m.newStationManager()
	passengerTrain := &m.PassengerTrain{
		Mediator: stationManager,
	}
	goodTrain := &m.GoodTrain{
		Mediator: stationManager,
	}
	passengerTrain.RequestArrival()
	goodTrain.RequestArrival()
	passengerTrain.Departure()
}
```  
## 26. Behavioral pattern - Memento
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172660-6677dce5-c3b9-47d5-9fb0-a3db397aac21.png"/>    
**Các thành phần**   
- Originator   
- Memento  
- Caretaker   

**File** *originator.go*
```Go
//File originator.go
package memento

import "fmt"

type Originator struct {
	State string
}

func (e *Originator) CreateMemento() *Memento {
	return &Memento {
		state: e.State
	}
}

func (e *Originator) RestoreMemento(m *Memento) {
	e.State = m.state
}

func (e *Originator) GetState() string {
	return e.State
}

func (e *Originator) SetState(state string) {
	e.State = state
}
```  

**File** *memento.go*
```Go
//File memento.go
package memento

type Memento struct {
	state string
}

func (m *Memento) GetSaveState() string {
	return m.state
}
```   
**File** *careTaker.go*
```Go
//File caretaker.go
package memento

type CareTaker struct {
	MementoArray []*Memento
}

func (c *CareTaker) AddMemento(m *Memento) {
	c.mementoArray = append(c.MementoArray, m)
}

func (c *CareTaker) GetMemento(index int) *Memento {
	return c.MementoArray[index]
}
```   
**File** *main.go*
```Go
//File main.go
package main

import "fmt"

func main() {
	careTaker := &m.CareTaker{
		MementoArray: make([]*m.Memento, 0),
	}
	originator := &m.Originator {
		state : "A",
	}
	fmt.Println("Originator current state: %s\n", originator.GetState())
	careTaker.AddMemento(originator.CreateMemento())

	originator.SetState("B")
	fmt.Println("Originator current state: %s\n", originator.GetState())

	careTaker.AddMemento(originator.CreateMemento())
	originator.SetState("C")

	fmt.Println("Originator current state: %s\n", originator.GetState())
	careTaker.AddMemento(originator.CreateMemento())

	originator.RestoreMemento(careTaker.getMemento(1))
	fmt.Println("Originator current state: %s\n", originator.GetState())

	originator.RestoreMemento(careTaker.GetMemento(0))
	fmt.Println("Originator current state: %s\n", originator.GetState())
}
```   

## 27. Behavioral pattern - Observer
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172731-49a14b4f-756b-4d42-8932-2a2753c0a41e.png"/>    
**Các thành phần**     
- Subject Interface   
- Concrete Subject   
- Observe Interface 
- Concrete Observe   

**File** *subject.go*
```Go
//File subject.go
package observer

import "fmt"

type Subject interface {
	Register(observer Observer)
	Deregister(observer Observer)
	NotifyAll()
}
```   
**File** *item.go*
```Go
//File item.go
package observer

type Item struct {
	ObserveList []Observer
	Name string
	InStock bool
}

func NewItem(name string) *Item {
	return &Item {
		Name : name,
	}
}

func (i *Item) updateAvailability() {
	fmt.Printf("Item %s is now in stock\n", i.Name)
	i.InStock = true
	i.NotifyAll()
}

func (i *Item) Register(o Observer) {
	i.ObserveList = append(i.ObserveList, o)
}

func (i *Item) Deregister(o Observe) {
	i.ObserveList = removeFromSlice(i.ObserveList, o)
}

func (i *Item) Notify(o Observe) {
	for _, observer := range i.ObserveList {
		observer.Upadate(i.Name)
	}
}

func removeFromSlice(observerList []Observer, observeToRemove Observer) []Observe {
	l := len(observerList)
	for i, observer := range observerList {
		if observeToRemove.GetID() == observer.GetID() {
			observerList[l-1], observerList[i] = observerList[i], observerList[l-1]
			return observerList[:l-1]
		}
	}
	return observerList
}
```  
**File** *observer.go*
```Go
//File observer.go
package observer

type Observer interface {
	Update(string)
	GetID() string
}
```     
**File** *customer.go*
```Go
//File customer.go
package observer

import "fmt"

type Customer struct {
	ID string
}

func (c *Customer) Update(itemName string) {
	fmt.Prinf("Sending email to customer %s for item %s\n",c.ID, itemName)
}

func (c *Customer) GetID() string {
	return c.ID
}
```      
**File** *main.go*
```Go
//File main.go
package main

import o "observer"

func main() {
	shirtItem := o.NewItem("Nike shirt")
	observerFirst := &o.Customer(ID:"abc@gmail.com")
	observerSecond := &o.Customer(ID:"xyz@gmail.com")

	shirtItem.Register(observerFirst)
	shirtItem.Register(observerSecond)

	shirtItem.Deregister(observerSecond)
	shirtItem.updateAvailability()
}
```

## 28. Behavioral pattern - State
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172816-099490d0-a474-4f01-bf5c-2e5ecdaff30e.png"/>      
**Các thành phần:**   
- Context   
- State Interface   
- Concrete State ,...    
**File:** *machine.go*
```Go
//File machine.go
package state

import "fmt"
type Machine struct {
	current state
}

func NewMachine() *Machine {
	fmt.Println("Machine is ready")
	return &Machine{newOff()}
}

func (m *Machine) setCurrent(s state) {
	m.current.current = s
}

func (m *Machine) On() {
	m.current.on()
}

func (m *Machine) Off() {
	m.current.off()
}
```   

**File:** *state.go*
```Go
//File state.go
package state

type state interface {
	on(m *Machine)
	off(m *Machine)
}
```   
**File:** *on.go*
```Go
//File on.go
package state

import "fmt"

type on struct {
}

func newOn() state {
	return &on{}
}

func (o *on) on(m *Machine) {
	fmt.Println("Machine is already ON")
}

func (o *on) off(m *Machine) {
	fmt.Println("Machine is going from ON to OFF")
	m.setCurrent(newOff())
}
```    
**File:** *off.go*
```Go
//File off.go
package state

type off struct{
}

func newOff() state {
	return &off{}
}

func (o *off) on(m *Machine) {
	fmt.Println("Machine is going from OFF to ON")
	m.setCurrent(newOn())
}

func (o *on) off(m *Machine) {
	fmt.Println("Machine is already OFF")
}
```     
**File:** *main.go*
```Go
//File main.go
package main

import s "state"

func main() {
	machine := s.NewMachine()
	machine.Off()
	machine.On()
	machine.On()
	machine.Off()
}
```


## 29. Behavioral pattern - Strategy
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172858-2a8e5e59-f3a1-4d5a-9755-c52994304035.png"/>       
**Các thành phần:**    
- Context
- Strategy interface   
- Concrete Strategy,...   
**File** *cache.go*    
```Go
//File cache.go
package strategy

import "fmt"

type Cache struct {
	storage map[string] string
	evictionAlgo EvictionAlgo
	capacity int
	maxCapacity int
}

func InitCache(e EvictionAlgo) *Cache {
	storage := make(map[string]string)
	return &Cache {
		storage: storage,
		evictionAlgo: e,
		capacity: 0,
		maxCapacity: 2,
	}
}

func (c *Cache) SetEvictionAlgo(e EvictionAlgo) {
	e.evictionAlgo = e
}

func (c *Cache) Add(key, value string) {
	if c.capacity == c.maxCapacity {
		c.evictionAlgo.evict()
		c.capacity--
	}
	c.capacity++
	c.storage[key] = value
}
```   
**File** *evictionAlgo.go*    
```Go
//File evictionAlgo.go
package strategy

type EvictionAlgo interface {
	evict(c *Cache)
}
```  
**File** *fifo.go*    
```Go
//File fifo.go
package strategy

import "fmt"

type FIFO struct {}

func (f *FIFO) evict(c *Cache) {
	fmt.Println("Evicting by FIFO strategy")
}
```   
**File** *lru.go*    
```Go
//File lru.go
package strategy

import "fmt"

type LRU struct {}

func (l *LRU) evict(c *Cache) {
	fmt.Println("Evicting by LRU strategy")
}
```    
**File** *lfu.go*    
```Go
//File lfu.go
package strategy

import "fmt"

type LFU struct {}

func (f *LFU) evict(c *Cache) {
	fmt.Println("Evicting by LFU strategy")
}
```   
**File** *main.go*    
```Go
//File main.go
package main

import s "strategy"

func main() {
	lfu := &s.lfu{}
	cache := s.InitCache{lfu}
	cache.Add("a","1")
	cache.Add("b","2")
	cache.Add("c","3")
	lru := &s.LRU{}
	cache.SetEvictionAlgo(lru)
	cache.Add("d","4")
	fifo := &s.FIFO{}
	cache.SetEvictionAlgo(fifo)
	cache.Add("e","5")
}
```


## 30. Behavioral pattern - Template method
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172894-d24b2443-2cad-4ac8-b422-8a60a47be891.png"/>      
- **Các thành phần:**   
    
	- Method interface
	- Concrete Object,...
	- Template methods    
    
**File** *iOTP.go*
```Go
//File iOTP.go
package templateMethod

type IOtp interface {
	genRandomOTP(int) string
	saveOTPCache(string)
	getMessage(string) string
	sendNotification(string) error
}
```    
**File** *sms.go*
```Go
//File sms.go
package templateMethod

import "fmt"

type SMS struct {}

func (s *SMS) genRandomOTP(len int ) string {
	randomOTP := "1234"
	fmt.Printf("SMS: generating random OTP %s \n", randomOTP)
	return randomOTP
}

func (s *SMS) saveOTPCache(otp string) {
	fmt.Printf("SMS: saving OTP: %s to cache \n",otp)
}

func (s *SMS) getMessage(otp string) string{
	return "SMS OTP for login is " + otp
}

func (s *SMS) sendNotification(message string) error {
	fmt.Printf("SMS: sending SMS %s\n", message)
	return nil
}
```     
**File** *email.go*
```Go
//File email.go
package templateMethod

import "fmt"

type Email struct {}

func (e *email) genRandomOTP(len int ) string {
	randomOTP := "abcd"
	fmt.Printf("EMAIL: generating random OTP %s \n", randomOTP)
	return randomOTP
}

func (e *Email) saveOTPCache(otp string) {
	fmt.Printf("EMAIL: saving OTP: %s to cache \n",otp)
}

func (e *Email) getMessage(otp string) string{
	return "EMAIL OTP for login is " + otp
}

func (e *Email) sendNotification(message string) error {
	fmt.Printf("EMAIL: sending SMS %s\n", message)
	return nil
}
```      
**File** *otp.go*
```Go
//File otp.go
package templateMethod

type OTP struct {
	ObjectOTP IOtp
}

func (e *OTP) GenAndSendOTP (otplength int) error {
	otp := o.ObjectOTP.genRandomOTP(otplength)
	o.ObjectOTP.saveOTPCache(otp)
	message := o.ObjectOTP.getMessage(otp)
	err := o.ObjectOTP.sendNotification(message)
	if err != nil {
		return err
	}
	return nil
}
```     
**File** *main.go*
```Go
//File main.go
package main

import t "templateMethod"

func main() {
	smsOTP := &t.SMS{}
	o := &t.OTP {
		ObjectOTP : sendOTP,
	}
	o.GenAndSendOTP(4)

	emailOTP := &t.Email{}
	o = &t.OTP {
		ObjectOTP: emailOTP,
	}
	o.GenAndSendEmail(4)
}
```

## 31. Behavioral pattern - Visitor
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172937-a6129e6b-e761-4cb4-a38a-c22ca6fc9828.png"/>     

## 32. Structural pattern - Adapter
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143172968-b0a5e113-7e47-406a-ad99-bb1b4066e55a.png"/>     

## 33. Structural pattern - Bridge
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173052-9eb02bd9-d3c1-4a80-841c-019d3c40c33f.png"/>     

## 34. Structural pattern - Composite
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173116-ae46b30c-d050-4cbf-8c63-90ee0ed6cc47.png"/>    

## 35. Structural pattern - Facade
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173146-7befaac8-d5f9-43c3-a343-56bddb9efb53.png"/>   

## 36. Structural pattern - Flyweight
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173182-94c720c9-9728-45f9-a39e-6b02cd50cc10.png"/>      

## 37. Structural pattern - Proxy  
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173227-6366f90d-1d29-45bb-b4ad-d3887e67e0ef.png"/>     

## 38. Structural pattern - Decorator
<img  width="100%" align="center" src="https://user-images.githubusercontent.com/68103697/143173290-810ceef3-b11d-4366-a929-2c684a6d0ced.png"/>     

## 39. Stack & Heap & Cấp phát bộ nhớ











