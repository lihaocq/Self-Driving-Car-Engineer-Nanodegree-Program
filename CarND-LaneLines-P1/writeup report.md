# **Finding Lane Lines on the Road** 

#### The goals of this document are the following:

1. Describe the pipeline for the project of Finding Lane Lines on Road. Make sure important points is documented properly.
2. Reflect on this pipeline to find the shortcomings with current pipeline and suggest possible improvements to this pipeline.



---

#### Describe the pipeline  for Finding Lane Lines on the Road

- **Get grey version of image**: for embedded system, it's space resource is limited, so images should be processed in 'Greyscale' version. The following command transfer a image from 'Blue, green, red' version to gray version.

```python 3
grey_image = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

- **Smoothing image**: image processed by `cv2.cvtColor` still contains noises, edges and so on. These undesirable stuff should be removed before further processing by blurring. Image blurring is achieved by convolving the image with a low-pass filter kernel. High frequency content is removed from the image. So edges are blurred a little bit in this operation. A most popular blurring method is Gaussian blur.

  ```python 3
  blur_gray = cv2.GaussianBlur(grey_image,(kernel_size, kernel_size),0)
  ```

  Basic idea of blurring is to use average value of adjacent pixels as the value of central pixel. Kernel size the the size of adjacent pixels to be averaged. A good value is 5.

-  **Edge Detection (Canny Edge Detection)**: The basic theory behind edge detection is, wherever there is an edge, the pixel on either side of the edge have a big difference (also called gradient) between their intensities. First the input image is scanned in both horizontal and vertical direction to find gradient for each pixel. After getting gradient magnitude and direction, a full scan of image is done to remove any unwanted pixels which may not constitute the edge. For this, at every pixel, pixel is checked if it is a local maximum in its neighborhood. 

  We need two threshold values, minVal and maxVal. Any edges with intensity gradient more than maxVal are sure to be edges and those below minVal are sure to be non-edges, so discarded. Those who lie between these two thresholds are classified edges or non-edges based on their connectivity. If they are connected to “sure-edge” pixels, they are considered to be part of edges. Otherwise, they are also discarded. 

```python 3
    # to find edges of image by Canny edge detection method
    low_threshold = 50
    high_threshold = 150
    edges = cv2.Canny(blur_gray, low_threshold, high_threshold)
```

- **Region selection**:  

  One thing to consider is that we do not want to find edges in all the whole image. We are only interested in a trapezoidal area where the lane lines are.

  ```python 3
      ##### Next we'll create a masked edges image using cv2.fillPoly()
      mask = np.zeros_like(edges)   #Return an array of zeros with the same shape and type as a given array.
      ignore_mask_color = 255  
  
      imshape = image.shape
  
      # bottom left, top left, top right, bottom right
      vertices = np.array([[(0,imshape[0]),(450, 320), (500, 320), (imshape[1],imshape[0])]], dtype=np.int32) #generate an array
      # Fills the area (mask) bounded by one or more polygons (vertices).
      cv2.fillPoly(mask, vertices, ignore_mask_color)
      # mask the area (edges) with prepared mask area (mask)
      masked_edges = cv2.bitwise_and(edges, mask)
  ```

- **Detect line by Hough Line Transform**: Hough’s transform is only used to detect straight line instead of curve or circular edges. Referring to the lecture, we know that a line in x-y space is a point in Hough space. A point in x-y space is a sine wave in Hough space. So we first transform all points in x-y space to Hough space then look for the intersections. These intersections are lines in x-y space.

  ```python 3
  ##### detect lines by Hough transform
      rho = 1 # distance resolution in pixels of the Hough grid
      theta = np.pi/180 # angular resolution in radians of the Hough grid
      threshold = 15     # minimum number of votes (intersections in Hough grid cell)
      min_line_length = 40 #minimum number of pixels making up a line
      max_line_gap = 30    # maximum gap in pixels between connectable line segments
      line_image = np.copy(image)*0 # creating a blank to draw lines on
      
      # Output “lines” is an array containing endpoints of detected line segments.
      lines = cv2.HoughLinesP(masked_edges, rho, theta, threshold, np.array([]),min_line_length, max_line_gap)
  ```

  

- **Extrapolate lines and draw lines on image**: Extrapolating the lines to the top and bottom of the trapezoidal area and then plot the extrapolated lines to image.

  ```python 3
      ##### extrapolate “lines” and draw lines on a blank image.
      ine_image = np.copy(image)*0 # creating a blank to draw lines on
      # extrapolate lines segments and draw lines on a (masked) blank image.
      draw_lines_final(ine_image, lines, vertices)
      
      # Draw the lines on the original image and return it.
      lines_edges = cv2.addWeighted(image, 0.8, line_image, 1, 0)
  ```




* **How to modify the draw_lines() function**

  Before using this draw_lines() function, in the trapezoidal area we find two set of lines, one with negative slope and the other with positive slope. The lines with negative slope are the left lane line and the others with positive slope is right lane line. The goal of this modified function is to merge the set of lines into one line and extrapolate the line to the bottom and top of the trapezoidal area. To merge the set of lines, I used some kind of averaging depends on the length of each line. For details, check the following code.

  ```python 3
  def draw_lines_final(img, lines,vertices, color=[255, 0, 0], thickness=10):
     
      # extrapolate lines segments and draw lines on a image.
     
      # bottom left, top left, top right, bottom right
      l_bottom = vertices[0][0]
      l_top = vertices[0][1]
      r_bottom = vertices[0][2]
      r_top = vertices[0][3]   
      
      
      l_slope = np.array([])
      l_num =0
      l_length = np.array([])
      l_x1=np.array([])
      l_x2=np.array([])
      l_y1=np.array([])
      l_y2=np.array([])
      
      r_slope = np.array([])
      r_num =0
      r_length = np.array([])
      r_x1=np.array([])
      r_x2=np.array([])
      r_y1=np.array([])
      r_y2=np.array([]) 
      
      
      for line in lines:
          for x1,y1,x2,y2 in line:
              slope = (y2-y1)/(x2-x1)
              line_length = np.sqrt((y2-y1)**2+(x2-x1)**2)
              
              if slope<0:
              
                  l_num += 1
                  l_slope= np.append(l_slope,slope)
                  l_length= np.append(l_length,line_length)
                  
                  l_x1=np.append(l_x1,x1)
                  l_x2=np.append(l_x2,x2)
                  l_y1=np.append(l_y1,y1)
                  l_y2=np.append(l_y2,y2)
                  
                  
              elif slope > 0:
                  r_num += 1
                  r_slope=np.append(r_slope,slope)
                  r_length=np.append(r_length,line_length)
  
                  r_x1=np.append(r_x1,x1)
                  r_x2=np.append(r_x2,x2)
                  r_y1=np.append(r_y1,y1)
                  r_y2=np.append(r_y2,y2)
                  
      l_avr_slope = np.mean(l_slope)
      r_avr_slope = np.mean(r_slope)
      
      
      sum_l_x1=0
      for i in range(0,l_num):
          sum_l_x1 += l_x1[i]*l_length[i]
      avr_l_x1=sum_l_x1/np.sum(l_length)
  
      sum_l_x2=0
      for i in range(0,l_num):
          sum_l_x2 += l_x2[i]*l_length[i]
      avr_l_x2=sum_l_x2/np.sum(l_length)
  
      sum_l_y1=0
      for i in range(0,l_num):
          sum_l_y1 += l_y1[i]*l_length[i]
      avr_l_y1=sum_l_y1/np.sum(l_length)
  
      sum_l_y2=0
      for i in range(0,l_num):
          sum_l_y2 += l_y2[i]*l_length[i]
      avr_l_y2=sum_l_y2/np.sum(l_length)
      
      
      l_m = l_avr_slope
      l_b = avr_l_y1 - l_m*avr_l_x1
      
      l_final_x1 = (l_bottom[1]-l_b)/l_m
      l_final_y1 = l_bottom[1]
      
      l_final_x2 = (l_top[1]-l_b)/l_m
      l_final_y2 = l_top[1]    
      
      
  
      sum_r_x1=0
      for i in range(0,r_num):
          sum_r_x1 += r_x1[i]*r_length[i]
      avr_r_x1=sum_r_x1/np.sum(r_length)
  
      sum_r_x2=0
      for i in range(0,r_num):
          sum_r_x2 += r_x2[i]*r_length[i]
      avr_r_x2=sum_r_x2/np.sum(r_length)
  
      sum_r_y1=0
      for i in range(0,r_num):
          sum_r_y1 += r_y1[i]*r_length[i]
      avr_r_y1=sum_r_y1/np.sum(r_length)
  
      sum_r_y2=0
      for i in range(0,r_num):
          sum_r_y2 += r_y2[i]*r_length[i]
      avr_r_y2=sum_r_y2/np.sum(r_length)
  
      
      r_m = r_avr_slope
      r_b = avr_r_y1 - r_m*avr_r_x1
      
      r_final_x1 = (r_bottom[1]-r_b)/r_m
      r_final_y1 = r_bottom[1]
      
      r_final_x2 = (r_top[1]-r_b)/r_m
      r_final_y2 = r_top[1]    
      
  
      cv2.line(img, (int(l_final_x1), int(l_final_y1)), (int(l_final_x2), int(l_final_y2)), color, thickness)
      cv2.line(img, (int(r_final_x1), int(r_final_y1)), (int(r_final_x2), int(r_final_y2)), color, thickness)
              
  ```

  







---

#### Reflection

**1. Shortcomings with this pipeline**

* Current pipeline lacks ability to detect curve and circular edges in image. Hough Line Transform can only detect straight lines in image. It can not handle curve and circular edges. However, on real roads, some lane lines are with very large curvature, which is out the ability of my pipe line.

**2. Suggest possible improvements to your pipeline**

* use other algorithm rather than Hough’s transform to detect straight lane lines and lines with larger curvature.