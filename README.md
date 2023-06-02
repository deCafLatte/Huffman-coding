# Huffman-coding

根据课程设计要求完成对哈夫曼编码的实现

Compression：

1. 构建频率字典
2. 使用MinHeap构建优先级队列
3. 通过选择2个min节点并合并它们来构建HuffmanTree
4. 为字符分配代码(通过从根遍历树)
5. 对输入文本进行编码(通过将字符替换为其代码)
6. 如果最终编码位流的总长度不是8的倍数，则在文本中添加一些填充
7. 将这个填充信息(以8位为单位)存储在整个编码比特流的开始处
8. 将结果写入输出二进制文件

```python
import heapq
import os

class HuffmanCoding:
	def __init__(self, path):
		self.path = path
		self.heap = []#存最小堆
		self.codes = {}#存对应的哈夫曼编码
		self.reverse_mapping = {}

	class HeapNode:
		def __init__(self, char, freq):
			self.char = char
			self.freq = freq
			self.left = None
			self.right = None

		# defining comparators less_than and equals
		def __lt__(self, other):
			return self.freq < other.freq

		def __eq__(self, other):
			if(other == None):
				return False
			if(not isinstance(other, HeapNode)):
				return False
			return self.freq == other.freq

	# functions for compression:
#计算频率并返回
	def make_frequency_dict(self, text):
		frequency = {}
		for character in text:
			if not character in frequency:
				frequency[character] = 0
			frequency[character] += 1
		return frequency
'''是对给定的文本进行遍历，并统计每个字符在文本中出现的频率。
它会返回一个字典，其中键是文本中的字符，值是对应字符的频率。
'''
#make priority queue
	def make_heap(self, frequency):
		for key in frequency:
			node = self.HeapNode(key, frequency[key])
			heapq.heappush(self.heap, node)
'''
遍历频率字典中的每个字符，为每个字符创建一个节点，并将节点按照频率的顺序插入到
最小堆 self.heap 中。这样，self.heap 列表中的元素将按照频率从小到大排列，满足最小堆的性质。
'''
#build huffman tree Save root node in heap构建 Huffman 树，并将根节点保存回最小堆。
	def merge_nodes(self):
		while(len(self.heap)>1):
			node1 = heapq.heappop(self.heap)
			node2 = heapq.heappop(self.heap)

			merged = self.HeapNode(None, node1.freq + node2.freq)
			merged.left = node1
			merged.right = node2

			heapq.heappush(self.heap, merged)
'''
在最小堆中循环执行以下操作：弹出频率最小的两个节点，合并它们创建一个新节点，将新节点插入回最小堆中
这个过程不断重复，直到最小堆中只剩下一个节点，这个节点就是 Huffman 树的根节点。
'''

#递归地生成字符的编码，并将编码保存在类的属性中
	def make_codes_helper(self, root, current_code):
		if(root == None):
			return

		if(root.char != None):
			self.codes[root.char] = current_code
#将当前字符的编码 current_code 存储在 self.codes 字典中，以字符为键，编码为值。
			self.reverse_mapping[current_code] = root.char
#将当前编码 current_code 存储在 self.reverse_mapping 字典中，以编码为键，字符为值。
			return

		self.make_codes_helper(root.left, current_code + "0")
'''
递归调用 make_codes_helper 方法，传递当前节点的左子节点 root.left 和添加 "0" 后的
当前编码 current_code。这表示在左子树上继续生成编码，将当前编码加上 "0"。
'''
		self.make_codes_helper(root.right, current_code + "1")
'''
递归地遍历 Huffman 树的每个节点，根据路径上的左右走向来生成每个字符的编码。
编码信息将存储在 self.codes 字典中，以字符为键，编码为值。
同时，也将编码信息存储在 self.reverse_mapping 字典中，以编码为键，字符为值，用于后续的解码操作。
'''

#make codes for characters and save生成字符的编码并保存
	def make_codes(self):
		root = heapq.heappop(self.heap)
		current_code = ""
		self.make_codes_helper(root, current_code)
'''
从最小堆中弹出 Huffman 树的根节点，并通过调用 make_codes_helper 方法开始生成每个字符的编码
'''

#replace characters with code and return 将文本中的字符替换为对应的编码
	def get_encoded_text(self, text):
		encoded_text = ""
		for character in text:
			encoded_text += self.codes[character]
		return encoded_text
'''
遍历给定的文本字符串，并根据之前生成的编码字典，将每个字符替换为其对应的编码。
最终，返回替换后的编码文本字符串。
'''

#pad encdede text and  return 填充文本
	def pad_encoded_text(self, encoded_text):
		extra_padding = 8 - len(encoded_text) % 8
		for i in range(extra_padding):
			encoded_text += "0"

		padded_info = "{0:08b}".format(extra_padding)
		encoded_text = padded_info + encoded_text
		return encoded_text
'''
根据编码后的文本的长度，计算需要额外填充的位数，然后将相应数量的 "0" 字符追加到编码文本的末尾，
最后将填充位数信息添加到编码文本的开头。填充后的编码文本长度将是8的倍数，以确保数据能够正确地进行解压缩。
'''

#convert bits into bytes.Return byte array 将填充后的编码文本转换为字节数组
	def get_byte_array(self, padded_encoded_text):
		if(len(padded_encoded_text) % 8 != 0):
			print("Encoded text not padded properly")
			exit(0)

		b = bytearray()
		for i in range(0, len(padded_encoded_text), 8):
			byte = padded_encoded_text[i:i+8]
			b.append(int(byte, 2))
		return b
'''
将填充后的编码文本划分为8个字符一组，每组表示一个字节的二进制数据。
然后将每个字节转换为整数，并将其作为字节添加到字节数组中。
最终返回转换后的字节数组，以便进行文件的写入或其他操作。
'''

	def compress(self):
		filename, file_extension = os.path.splitext(self.path)
#将路径分割为文件名和扩展名，并将它们分别赋值给 filename 和 file_extension 变量。

		output_path = filename + ".bin"

		with open(self.path, 'r+') as file, open(output_path, 'wb') as output:
'''打开指定路径（self.path）的文件，并将其赋值给名为 file 的变量。这里使用了模式参数 'r'，
表示以只读模式打开文件。通过这个文件对象，可以读取文件中的内容。'''

'''打开指定路径（output_path）的文件，并将其赋值给名为 output 的变量。
这里使用了模式参数 'wb'，表示以二进制写入模式打开文件。
通过这个文件对象，可以将数据以二进制形式写入文件。'''
			text = file.read()
			text = text.rstrip() #去除末尾空格

			frequency = self.make_frequency_dict(text)
'''
调用了 make_frequency_dict 方法，该方法接受一个文本字符串作为参数，
并返回一个字典 frequency，其中包含文本中每个字符及其对应的频率。这个字典将用于构建 Huffman 树。
'''
			self.make_heap(frequency)
'''
这行代码调用了 make_heap 方法，该方法接受之前生成的频率字典 frequency 作为参数，
并使用频率信息构建一个最小堆。
'''
			self.merge_nodes()
'''调用了 merge_nodes 方法，该方法执行 Huffman 编码算法的合并节点步骤。
在这一步骤中，根据最小堆中的节点频率，不断合并两个具有最小频率的节点，直到只剩下一个根节点。
这个根节点就是 Huffman 树的根。
'''
			self.make_codes()
'''调用了 make_codes 方法，该方法根据构建的 Huffman 树生成每个字符的编码。
编码是由 0 和 1 组成的比特串，它们表示字符在 Huffman 树中的路径。
'''
#这几行代码实现了 Huffman 编码算法的关键步骤，包括生成字符频率字典、
#构建最小堆、合并节点生成 Huffman 树，以及生成字符编码。这些步骤将用于压缩和解压缩文本数据。
			encoded_text = self.get_encoded_text(text)
			padded_encoded_text = self.pad_encoded_text(encoded_text)

			b = self.get_byte_array(padded_encoded_text)
			output.write(bytes(b))

		print("Compressed")
		return output_path

	""" functions for decompression: """

	def remove_padding(self, padded_encoded_text):
		padded_info = padded_encoded_text[:8]
		extra_padding = int(padded_info, 2)

		padded_encoded_text = padded_encoded_text[8:] 
		encoded_text = padded_encoded_text[:-1*extra_padding]

		return encoded_text

	def decode_text(self, encoded_text):
		current_code = ""
		decoded_text = ""

		for bit in encoded_text:
			current_code += bit
			if(current_code in self.reverse_mapping):
				character = self.reverse_mapping[current_code]
				decoded_text += character
				current_code = ""

		return decoded_text

	def decompress(self, input_path):
		filename, file_extension = os.path.splitext(self.path)
		output_path = filename + "_decompressed" + ".txt"

		with open(input_path, 'rb') as file, open(output_path, 'w') as output:
			bit_string = ""

			byte = file.read(1)
			while(len(byte) > 0):
				byte = ord(byte)
				bits = bin(byte)[2:].rjust(8, '0')
				bit_string += bits
				byte = file.read(1)

			encoded_text = self.remove_padding(bit_string)

			decompressed_text = self.decode_text(encoded_text)
			
			output.write(decompressed_text)

		print("Decompressed")
		return output_path
			

```
