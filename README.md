# 1. Tổng quan PicoSoC và vai trò của từng khối

PicoSoC là một thiết kế SoC mẫu xây quanh lõi PicoRV32 được kết nối với bộ nhớ, flash, UART và các ngoại vi khác để thành một hệ thống có thể nạp lên FPGA và chạy firmware thực tế.

![PicoSoC](image.png)

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Các khối cốt lõi của PicoSoC</strong></p>
<blockquote>
<p><strong>•</strong> PicoRV32 CPU: thực thi chương trình và phát sinh truy cập bộ nhớ/ngoại vi.</p>
<p><strong>•</strong> Internal SRAM: scratchpad nhỏ để chứa stack, biến tạm và dữ liệu truy cập nhanh.</p>
<p><strong>•</strong> SPI Memory I/O + external SPI flash: nơi chứa firmware chính và logic giúp CPU đọc flash như một vùng nhớ.</p>
<p><strong>•</strong> UART: giao tiếp nối tiếp phục vụ debug, in thông báo và nhận lệnh cơ bản.</p>
<p><strong>•</strong>Vùng ngoại vi: khu vực để người thiết kế mở rộng: LED, GPIO, timer, custom IP, accelerator.</p>
</blockquote></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 1.1. Khối CPU

CPU trong PicoSoC là PicoRV32. Nó không truy cập ngoại vi bằng các lệnh đặc biệt mà chỉ đọc/ghi vào các địa chỉ nhất định trong memory map. Phía sau những địa chỉ đó là mạch giải mã để nối CPU đến SRAM, flash controller, UART hay ngoại vi do người dùng thêm vào.

## 1.2. Khối SRAM 

SRAM trong PicoSoC có vai trò scratchpad (lưu trữ dữ liệu tạm ở vùng nhớ nhỏ để xử lý nhanh). Dung lượng mặc định nhỏ, không nhằm thay thế hoàn toàn cho bộ nhớ chương trình, mà để cung cấp nơi lưu stack và dữ liệu tạm có tốc độ truy cập cao hơn flash.

## 1.3. Khối SPI flash và SPI Memory I/O

Flash ngoài là nơi chứa firmware. Khối SPI Memory I/O đóng vai trò cầu nối: nó nhận truy cập bộ nhớ từ CPU, chuyển thành tín hiểu SPI rồi trả dữ liệu về từ flash. Nhờ đó CPU có thể đọc lệnh và dữ liệu trong flash gần giống như đang đọc từ một vùng memory-mapped thông thường.

Ngoài ra còn có thanh ghi hỗ trợ các chế độ tăng tốc truy cập flash như QSPI, DDR và CRM. Trong đó, QSPI (Quad SPI) cho phép truyền dữ liệu qua nhiều đường I/O hơn để tăng băng thông, DDR (Double Data Rate) tận dụng cả hai cạnh xung clock để truyền dữ liệu nhanh hơn, còn CRM (Continuous Read Mode) cho phép đọc liên tiếp nhiều dữ liệu mà không phải lặp lại toàn bộ phần khởi tạo của mỗi lần đọc. Bên cạnh đó, trường Read latency (dummy cycles) dùng để cấu hình số chu kỳ chờ cần thiết để flash chuẩn bị dữ liệu trước khi xuất ra bus. Việc thiết lập đúng các chế độ này giúp tăng tốc độ đọc firmware từ flash và cải thiện hiệu năng của toàn hệ thống.

## 1.4. Khối UART

UART là ngoại vi cơ bản nhưng rất hữu ích trong SoC. CPU chỉ cần ghi vào thanh ghi data để gửi một byte, hoặc đọc từ thanh ghi đó để nhận byte mới nhất. Đồng thời có một thanh ghi divider dùng để xác lập baud rate theo clock hệ thống.

## 1.5. Vùng ngoại vi

Phần địa chỉ từ 0x03000000 trở đi được dành cho các ngoại vi do người dùng mở rộng. Đây là nơi rất thuận lợi để gắn LED, GPIO, timer, PWM hoặc các khối tăng tốc mật mã.

# 2. Memory map và cơ chế overlay

Memory map là cách SoC quy định mỗi dải địa chỉ thuộc về khối nào. Với PicoSoC, tổ chức địa chỉ rất gọn nên nhìn vào memory map có thể suy ra gần như toàn bộ luồng hoạt động của hệ thống.

**Bảng 1. Memory map chính của PicoSoC**

| **Dải địa chỉ**         | **Khối**                        | **Chức năng**                                                                                              |
|-------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------|
| 0x00000000 – 0x00FFFFFF | Internal SRAM / vùng đầu bộ nhớ | SRAM nội bộ dung lượng nhỏ được overlay lên đầu không gian nhớ; dùng làm scratchpad, stack và dữ liệu tạm. |
| 0x01000000 – 0x01FFFFFF | External Serial Flash           | Ánh xạ trực tiếp bộ nhớ flash SPI ngoài; firmware chính thường nằm tại đây.                                |
| 0x02000000 – 0x02000003 | SPI Flash Config                | Thanh ghi cấu hình bộ điều khiển flash: MEMIO, QSPI, DDR, CRM, dummy cycles, bit-bang IO.                  |
| 0x02000004 – 0x02000007 | UART Divider                    | Thiết lập tần số baud bằng cách chia clock hệ thống.                                                       |
| 0x02000008 – 0x0200000B | UART Data                       | Ghi để gửi byte qua UART, đọc để nhận byte hoặc kiểm tra buffer nhận.                                      |
| 0x03000000 – 0xFFFFFFFF | Ngoại vi                        | Vùng ngoại vi do người thiết kế mở rộng: LED, GPIO, timer, custom accelerator, giao tiếp chuyên dụng.      |

## 2.2. Overlay

Overlay có thể hiểu là một vùng nhớ được đặt chồng lên trên vùng nhớ khác trong cùng không gian địa chỉ. Cụ thể hơn, SRAM được overlay lên đầu vùng flash. Vì thế khi CPU truy cập vào những địa chỉ đầu tiên, nó sẽ nhìn thấy SRAM trước; chỉ khi vượt ra ngoài phần SRAM vật lý thì dữ liệu mới được đọc từ flash ở địa chỉ tương ứng.

Cách tổ chức này giúp phần mềm có một vùng đầu bộ nhớ trông giống “main memory”, trong đó phần nhỏ phía trước là SRAM nhanh và có thể ghi, còn phần lớn phía sau là flash lớn hơn nhưng chậm hơn và chủ yếu dùng để chứa chương trình.

# 3. Luồng boot và cách hệ thống vận hành

Khi hệ thống reset, PicoRV32 sẽ bắt đầu tại địa chỉ reset vector do thiết kế quy định. Trong PicoSoC, firmware được đặt trong flash với layout tương ứng để CPU có thể fetch lệnh ngay từ đầu quá trình khởi động. Từ góc nhìn của CPU, nó chỉ đang đọc instruction từ một địa chỉ nhớ; còn phía dưới là khối SPI Memory I/O thực hiện toàn bộ giao tiếp SPI với flash.

> **•** Bước 1: nạp bitstream FPGA và image firmware.
>
> **•** Bước 2: reset hệ thống, CPU nhảy tới reset vector.
>
> **•** Bước 3: CPU fetch lệnh từ flash thông qua SPI Memory I/O.
>
> **•** Bước 4: firmware khởi tạo stack, UART và các biến cần thiết.
>
> **•** Bước 5: chương trình chính chạy, điều khiển LED hoặc các ngoại vi memory-mapped khác.

Nếu cần tăng tốc flash, firmware có thể thiết lập cờ quad mode của chip nhớ rồi bật cấu hình QSPI ở phía controller. Nhờ vậy có thể tổ chức SoC một cách tối giản.

# 4. PicoRV32 - lõi xử lý bên trong PicoSoC

PicoRV32 là một lõi vi xử lý RISC-V 32-bit được tối ưu theo hướng nhỏ gọn, dễ tích hợp và đạt tần số hoạt động cao trên FPGA. Thay vì hướng tới CPI (Cycles per Instruction) rất thấp như các bộ xử lý pipeline lớn, PicoRV32 chấp nhận hiệu năng vừa phải để đổi lấy diện tích phần cứng nhỏ và khả năng đóng timing tốt.

Lõi này có thể cấu hình để hỗ trợ các biến thể như RV32I, RV32IM, RV32IC, RV32IMC hoặc RV32E. Điều đó cho phép người thiết kế lựa chọn giữa mức độ đầy đủ của tập lệnh và lượng tài nguyên phần cứng sử dụng phù hợp với các hệ thống nhúng nhỏ và các thiết kế tích hợp accelerator.

> **•** Có thể dùng bus native đơn giản hoặc các biến thể AXI4-Lite, Wishbone.
>
> **•** Có sẵn cơ chế PCPI để mở rộng lệnh bằng bộ đồng xử lý ngoài.
>
> **•** Có thể bật hệ thống ngắt đơn giản nhưng không hoàn toàn theo privileged ISA chuẩn của RISC-V.

# 5. Kiến trúc, giao tiếp và các cơ chế mở rộng của PicoRV32

## 5.1. Biến thể 

Các biến thể như:

> **•** picorv32: lõi CPU dùng giao tiếp bộ nhớ native.
>
> **•** picorv32_axi: lõi CPU với AXI4-Lite master interface.
>
> **•** picorv32_wb: lõi CPU với Wishbone master interface.
>
> **•** picorv32_axi_adapter: cầu nối từ native interface sang AXI4-Lite.
>
> **•** picorv32_pcpi_mul / div: các ví dụ đồng xử lý qua PCPI.

## 5.2. Native memory interface

PicoRV32 thông qua Native Memory Interface để giao tiếp và truy cập bộ nhớ, flash và các ngoại vi memory-mapped thông qua hàng loạt các tín hiệu.

> **•** mem_valid: CPU đang yêu cầu truy cập.
>
> **•** mem_ready: bộ nhớ đã sẵn sàng / đã xong.
>
> **•** mem_addr: địa chỉ truy cập.
>
> **•** mem_wdata và mem_wstrb: dữ liệu ghi và byte-enable.
>
> **•** mem_rdata: dữ liệu đọc trả về.
>
> **•** mem_instr: phân biệt truy cập instruction fetch với truy cập dữ liệu.

CPU phát mem_valid khi có yêu cầu, phía bộ nhớ hoặc ngoại vi phản hồi bằng mem_ready khi dữ liệu đã sẵn sàng hoặc thao tác ghi đã hoàn tất. Cách làm này rất dễ dùng để tự viết decoder cho ROM, RAM, UART, GPIO hoặc các ngoại vi tự thiết kế.

## 5.3. PCPI 

PCPI (Pico Co-Processor Interface) là một giao tiếp mở rộng của PicoRV32 cho phép hiện thực các lệnh không rẽ nhánh bằng bộ đồng xử lý bên ngoài. Do CPU PicoRV32 rất nhỏ, nên không nhất thiết tích hợp mọi phần cứng tính toán bên trong. Khi CPU gặp một lệnh không do lõi xử lý trực tiếp nhưng có hỗ trợ qua PCPI, nó sẽ đưa instruction cùng các toán hạng nguồn ra interface này. Đồng xử lý bên ngoài có thể nhận lệnh, tính toán rồi trả kết quả trở lại cho CPU. Cơ chế đó đặc biệt phù hợp với các đề tài tích hợp bộ tăng tốc AES, SHA3, Ascon hoặc các lệnh chuyên dụng do người thiết kế định nghĩa.

## 5.4. Hệ thống ngắt 

PicoRV32 hỗ trợ 32 đường ngắt. Khi có ngắt, lõi sẽ chuyển sang chương trình phục vụ ngắt và cung cấp thông tin cần thiết thông qua các q-register. Cụ thể, q0 lưu địa chỉ quay về sau khi xử lý ngắt, còn q1 lưu bitmask các IRQ (Interrupt request) đang chờ xử lý; nhờ đó một lần vào handler có thể xử lý nhiều nguồn ngắt cùng lúc. Hai thanh ghi q2 và q3 có thể dùng như thanh ghi tạm để lưu/phục hồi dữ liệu trong quá trình phục vụ ngắt.

Để thao tác với hệ thống ngắt, PicoRV32 cung cấp một số lệnh custom như getq và setq để trao đổi dữ liệu giữa q-register và thanh ghi thường, retirq để quay về từ interrupt, maskirq để đặt “mặt nạ” ngắt, waitirq để CPU chờ đến khi có ngắt, và timer để cấu hình bộ đếm sinh ra timer interrupt. Cách tiếp cận này giúp hiện thực IRQ với phần cứng gọn hơn so với việc hỗ trợ đầy đủ privileged architecture chuẩn của RISC-V.

Nhờ dùng q-register và tập lệnh custom, PicoRV32 giữ được thiết kế nhỏ gọn và dễ hiện thực trên FPGA. Tuy nhiên, đổi lại, cơ chế ngắt này không hoàn toàn tương thích với mô hình interrupt/CSR chuẩn của RISC-V privileged architecture, nên mức độ tương thích với phần mềm hệ thống chuẩn sẽ hạn chế hơn.

Bonus: GNU toolchain **bộ công cụ phần mềm** dùng để biên dịch và liên kết chương trình cho CPU RISC-V, ví dụ như gcc, binutils, assembler, linker, v.v.
